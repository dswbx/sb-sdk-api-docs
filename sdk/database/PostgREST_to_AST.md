# PostgREST Request → AST Translation

Translates a web-spec `Request` arriving at a PostgREST-compatible endpoint into the unified AST format defined in [AST.md](./AST.md).

---

## 1. Overview & Top-Level Flow

**Input:** Web-spec `Request` (URL, method, headers, body)
**Output:** AST JSON object

```
Request ──┬──→ parseRoute ────────┐
          ├──→ parseHeaders ──────┤
          ├──→ parseSelect ───────┤  Phase 1 (parallel, pure)
          ├──→ parseBody ─────────┤
          └──→ parseQueryParams ──┘
                                  │
                    ┌─────────────┘
                    ▼
          ┌──→ resolveType ───────────────┐
          ├──→ resolveFilters ────────────┤
          ├──→ resolveTransforms ─────────┤  Phase 2 (combines Phase 1 outputs)
          ├──→ resolveMeta ───────────────┤
          ├──→ resolveRpcParams ──────────┤
          └──→ resolveUpsertParams ───────┘
                                          │
                                          ▼
                                        AST
```

### Signature

```ts
async function requestToAst(req: Request): Promise<AST>
```

### Execution

```ts
async function requestToAst(req: Request): Promise<AST> {
  // Phase 1 — parallel extraction
  const [route, headers, select, body, queryParams] = await Promise.all([
    parseRoute(req),
    parseHeaders(req),
    parseSelect(req),
    parseBody(req),
    parseQueryParams(req),
  ])

  // Phase 2 — resolution (sequential where needed)
  const type     = resolveType(route, req.method, headers)
  const filters  = resolveFilters(queryParams, select.embeddedAliases)
  const transforms = resolveTransforms(queryParams, select.embeddedAliases)
  const meta     = resolveMeta(headers, queryParams)
  const rpc      = resolveRpcParams(route, req.method, queryParams, body)
  const upsert   = resolveUpsertParams(queryParams, headers)

  return assemble(type, route, headers, select, body, filters, transforms, meta, rpc, upsert)
}
```

---

## 2. Phase 1 Parsers

### 2.1 parseRoute

Extracts table/function name from URL pathname.

```ts
type RouteResult = {
  from?: string       // table or view name
  function?: string   // function name (RPC)
  isRpc: boolean
}
```

**Logic:**

1. Strip base path (e.g. `/rest/v1/`)
2. Split remainder on `/`
3. If first segment = `rpc` → `isRpc: true`, second segment = function name
4. Otherwise → first segment = table/view name

**Edge cases:**
- URL-encoded names: `decodeURIComponent` before use → `%22my%20table%22` → `"my table"`
- Trailing slashes: strip before parsing
- Quoted identifiers in path: PostgREST doesn't support them in URL path, but decode anyway

```
GET /rest/v1/products?select=id        → { from: "products", isRpc: false }
POST /rest/v1/rpc/calculate_discount   → { function: "calculate_discount", isRpc: true }
```

---

### 2.2 parseHeaders

Extracts schema, Prefer tokens, and Accept value from request headers.

```ts
type HeadersResult = {
  schema?: string
  preferTokens: PreferToken[]
  accept: string
}

type PreferToken =
  | { key: "count"; value: "exact" | "planned" | "estimated" }
  | { key: "resolution"; value: "merge-duplicates" | "ignore-duplicates" }
  | { key: "return"; value: "minimal" | "headers-only" | "representation" }
  | { key: "tx"; value: "commit" | "rollback" }
  | { key: "missing"; value: "default" | "null" }
  | { key: "handling"; value: "strict" | "lenient" }
  | { key: "max-affected"; value: number }
  | { key: "timezone"; value: string }
  | { key: "params"; value: "single-object" | "multiple-objects" }
```

**Schema resolution:**

| Method | Header |
|--------|--------|
| GET, HEAD | `Accept-Profile` |
| POST, PATCH, DELETE | `Content-Profile` |

From `PostgrestBuilder.then()`:
```ts
if (['GET', 'HEAD'].includes(this.method)) {
  this.headers.set('Accept-Profile', this.schema)
} else {
  this.headers.set('Content-Profile', this.schema)
}
```

**Prefer header tokenization:**

The `Prefer` header is a comma-separated list of tokens in `key=value` format. Multiple `Prefer` headers are concatenated.

```
Prefer: count=exact, resolution=merge-duplicates
Prefer: missing=default
→ [{ key:"count", value:"exact" }, { key:"resolution", value:"merge-duplicates" }, { key:"missing", value:"default" }]
```

Parsing: split on `,`, then split each token on `=`. Special case: `max-affected` value is numeric.

**Accept header:** preserved raw for Phase 2 cardinality/format resolution.

---

### 2.3 parseSelect

The most complex parser. Parses the `select` query parameter into structured select entries, join definitions, and a set of embedded aliases for Phase 2.

```ts
type SelectResult = {
  select: SelectEntry[]         // AST select array
  join: Record<string, JoinDef> // extracted join definitions
  embeddedAliases: Set<string>  // all embed alias names
}
```

**Input:** the `select` query parameter value (e.g. `id,name,categories(id,name)`)

#### Grammar (EBNF)

```ebnf
query          = node { "," node }
node           = star | spread | renamed_field | field
star           = "*"
spread         = "..." field_with_embed
renamed_field  = identifier ":" field
field          = field_with_embed | non_embed_field

field_with_embed = identifier [ modifiers ] "(" [ query ] ")"
modifiers        = "!" ( "inner" | "left" | hint [ "!" ( "inner" | "left" ) ] )
hint             = identifier

non_embed_field  = identifier [ json_path ] [ typecast ] [ aggregation ] [ typecast ]
json_path        = ( "->" | "->>" ) identifier { ( "->" | "->>" ) identifier }
typecast         = "::" identifier
aggregation      = "." agg_function "()"
agg_function     = "count" | "sum" | "avg" | "min" | "max"

identifier       = letter { letter } | '"' { any_char } '"'
letter           = [a-zA-Z0-9_]
```

#### Parse outputs → AST mapping

| Select syntax | AST select entry |
|---------------|------------------|
| `*` | `"*"` (or omit `select` — see Open Questions) |
| `id` | `"id"` |
| `desc:description` | `{ "desc": { "column": "description" } }` |
| `salary::text` | `{ "salary": { "cast": "text" } }` |
| `amount::numeric.sum()` | `{ "amount": { "preCast": "numeric", "aggregate": "sum" } }` |
| `price.avg()::int` | `{ "price": { "column": "price", "aggregate": "avg", "cast": "int" } }` |
| `count()` | `{ "row_count": { "aggregate": "count" } }` — alias required via rename |
| `metadata->theme->>color` | `{ "color": { "column": "metadata", "path": "$.theme.color" } }` |
| `categories(id,name)` | embed entry (see below) |
| `...director(first_name)` | embed entry with `spread: true` |

#### Embed entries → AST select + join

When the parser encounters `field(subquery)`, it:

1. Creates an embed select entry: `{ "field": { "select": [...subquery...] } }`
2. Creates a join entry: `"field": {}`
3. Records `"field"` in `embeddedAliases`

Modifier handling:

| Syntax | Join properties |
|--------|-----------------|
| `table(...)` | `"table": {}` (default left join) |
| `table!inner(...)` | `"table": { "type": "inner" }` |
| `table!left(...)` | `"table": {}` (explicit left, same as default) |
| `table!hint_col(...)` | `"table": { "hint": "hint_col" }` |
| `table!hint_col!inner(...)` | `"table": { "hint": "hint_col", "type": "inner" }` |
| `alias:table(...)` | `"alias": { "from": "table" }` |
| `alias:table!inner(...)` | `"alias": { "from": "table", "type": "inner" }` |

**Nested embeds:** `actors(name,roles(character,films(title)))` recursively produces nested embed entries and nested join definitions.

---

### 2.4 parseBody

Async — reads the request body stream.

```ts
type BodyResult = {
  values?: object | object[]  // for insert/update/upsert
  args?: object               // for RPC
  raw?: string                // non-JSON RPC body
}
```

**Logic:**

1. Only for POST, PATCH, DELETE (GET/HEAD have no body)
2. Read body as text, then:
   - If `Content-Type: application/json` → `JSON.parse()`
   - Array → bulk insert values: `{ values: [...] }`
   - Object → single insert/update values or RPC args: `{ values: {...} }` or `{ args: {...} }`
   - If other content type (text/xml/binary) → `{ raw: bodyText }` (RPC only)
3. Empty body → `{}`

**Disambiguation between `values` and `args`** happens in Phase 2 (`resolveRpcParams`), not here. `parseBody` returns the parsed payload without semantic interpretation.

---

### 2.5 parseQueryParams

Raw extraction of all URL query parameters. No interpretation — just structured access for Phase 2.

```ts
type QueryParamsResult = Map<string, string[]>
```

**Logic:**

```ts
function parseQueryParams(req: Request): QueryParamsResult {
  const url = new URL(req.url)
  const result = new Map<string, string[]>()
  for (const [key, value] of url.searchParams.entries()) {
    const existing = result.get(key) ?? []
    existing.push(value)
    result.set(key, existing)
  }
  return result
}
```

Multiple values for same key are preserved as array (AND semantics in PostgREST).

Reserved keys (interpreted by specific resolvers): `select`, `order`, `limit`, `offset`, `on_conflict`, `columns`.

---

## 3. Phase 2 Resolvers

### 3.1 resolveType

Determines AST `type` from HTTP method, route, and headers.

```ts
function resolveType(
  route: RouteResult,
  method: string,
  headers: HeadersResult
): ASTType
```

| HTTP Method | Route | Prefer header | AST type |
|-------------|-------|---------------|----------|
| GET | table | — | `query` |
| HEAD | table | — | `query` (+ `$meta.head: true`) |
| POST | table | — | `insert` |
| POST | table | `resolution=merge-duplicates` | `upsert` |
| POST | table | `resolution=ignore-duplicates` | `upsert` (+ `ignoreDuplicates: true`) |
| PATCH | table | — | `update` |
| DELETE | table | — | `delete` |
| GET | rpc/* | — | `rpc` |
| HEAD | rpc/* | — | `rpc` (+ `$meta.head: true`) |
| POST | rpc/* | — | `rpc` |

**Note:** `put` is an AST-only concept. PostgREST has no HTTP method that produces it. The translator never emits `type: "put"` — it exists for backends that want to express full-row replacement semantics directly in AST. PostgREST achieves this via upsert with all columns.

---

### 3.2 resolveFilters

Transforms query parameters into AST `where` objects (top-level and embedded).

```ts
function resolveFilters(
  queryParams: QueryParamsResult,
  embeddedAliases: Set<string>
): {
  where: object                           // top-level where
  embeddedWheres: Record<string, object>  // alias → where for embeds
}
```

**Logic:**

1. Skip reserved keys: `select`, `order`, `limit`, `offset`, `on_conflict`, `columns`
2. For each remaining `key=value` pair:
   a. If key contains `.` and prefix matches an embedded alias → route to embedded where
   b. If key = `or` → parse as `$or` group
   c. If key = `and` → parse as `$and` group
   d. If key = `not.or` → parse as `$not: { $or: [...] }`
   e. If key = `not.and` → parse as `$not: { $and: [...] }`
   f. Otherwise → parse as column filter

**Column filter parsing:**

Format: `column=operator.value` or `column=not.operator.value`

```
status=eq.active         → { "status": { "$eq": "active" } }
price=gt.100             → { "price": { "$gt": 100 } }
status=not.eq.active     → { "status": { "$not": { "$eq": "active" } } }
name=like.*phone*        → { "name": { "$like": "*phone*" } }
id=in.(1,2,3)            → { "id": { "$in": [1, 2, 3] } }
```

**Embedded filter routing:**

```
categories.active=eq.true   → embeddedWheres["categories"] = { "active": { "$eq": true } }
reviews.rating=gte.4        → embeddedWheres["reviews"] = { "rating": { "$gte": 4 } }
```

If key prefix matches an embedded alias, strip the prefix and add to that embed's where. Otherwise treat the dotted key as a column name (though PostgREST uses dot notation for embedded table filters).

**Edge case: column name = join alias** — join alias takes precedence. If a table has a column named `actors` and there's also a join alias `actors`, `actors=eq.null` means anti-join (check join alias), not column filter.

**`or` group parsing:**

```
or=(status.eq.active,featured.is.true)
```

Parses recursively — splits on `,` respecting parentheses nesting, then parses each sub-expression. Supports nested `and`/`or`:

```
or=(status.eq.active,and(price.gt.100,price.lt.500))
```

→ `{ "$or": [{ "status": { "$eq": "active" } }, { "$and": [{ "price": { "$gt": 100 } }, { "price": { "$lt": 500 } }] }] }`

Embedded `or`: `categories.or=(active.eq.true,featured.is.true)` → routes to embed's where.

**Multiple values for same column key** (AND semantics):

```
?price=gte.100&price=lte.500
→ { "price": { "$gte": 100, "$lte": 500 } }
```

---

### 3.3 resolveTransforms

Extracts order/limit/offset from query params, for both top-level and embedded resources.

```ts
function resolveTransforms(
  queryParams: QueryParamsResult,
  embeddedAliases: Set<string>
): {
  order?: OrderEntry[]
  limit?: number
  offset?: number
  embeddedTransforms: Record<string, { order?: OrderEntry[]; limit?: number; offset?: number }>
}
```

**Order parsing:**

```
order=price.asc.nullsfirst,name.desc
→ [
    { "column": "price", "direction": "asc", "nullsFirst": true },
    { "column": "name", "direction": "desc" }
  ]
```

Format: `column.direction[.nulls]` — comma-separated for multiple columns.

| Segment | Values | Default |
|---------|--------|---------|
| direction | `asc`, `desc` | `asc` |
| nulls | `nullsfirst`, `nullslast` | omitted |

From `PostgrestTransformBuilder.order()`:
```ts
`${column}.${ascending ? 'asc' : 'desc'}${
  nullsFirst === undefined ? '' : nullsFirst ? '.nullsfirst' : '.nullslast'
}`
```

**Embedded transforms:**

```
categories.order=name.asc     → embeddedTransforms["categories"].order
categories.limit=5            → embeddedTransforms["categories"].limit
categories.offset=10          → embeddedTransforms["categories"].offset
```

Key prefix matching: if key is `{alias}.order`, `{alias}.limit`, or `{alias}.offset` where alias ∈ embeddedAliases → route to embed.

**Limit/offset:** plain numeric values.

From `PostgrestTransformBuilder.range()`: range is converted to offset + limit — `range(from, to)` → `offset=from`, `limit=to-from+1`. The translator receives already-converted params.

---

### 3.4 resolveMeta

Builds the AST `$meta` object from headers and query params.

```ts
function resolveMeta(
  headers: HeadersResult,
  queryParams: QueryParamsResult
): object
```

**From Prefer tokens:**

| Token | AST $meta field |
|-------|-----------------|
| `count=exact\|planned\|estimated` | `count` |
| `missing=default` | `missing: "default"` |
| `handling=strict` | `handling: "strict"` |
| `tx=rollback` | `rollback: true` |
| `max-affected=N` | `maxAffected: N` |
| `timezone=America/Los_Angeles` | `timezone: "America/Los_Angeles"` |

Note: `handling=strict` is also implicitly set by `maxAffected` (see `PostgrestTransformBuilder.maxAffected()`).

**From Accept header:**

| Accept value | AST $meta |
|-------------|-----------|
| `application/json` | (default, no special meta) |
| `application/vnd.pgrst.object+json` | `cardinality: "one"` |
| `text/csv` | (transport format, not in AST) |
| `application/geo+json` | (transport format, not in AST) |
| `application/vnd.pgrst.plan+text; for="..."; options=...;` | `explain: { ... }` |
| `application/vnd.pgrst.plan+json; for="..."; options=...;` | `explain: { ... }` |

**Explain header parsing:**

From `PostgrestTransformBuilder.explain()`:
```ts
`application/vnd.pgrst.plan+${format}; for="${forMediatype}"; options=${options};`
```

Where options = pipe-separated flags: `analyze|verbose|settings|buffers|wal`

Parse → `$meta.explain`:
```json
{
  "explain": {
    "analyze": true,
    "verbose": false,
    "settings": true,
    "buffers": true,
    "wal": false
  }
}
```

**Cardinality resolution (from Accept + method):**

- `application/vnd.pgrst.object+json` → `cardinality: "one"`
- `application/json` with GET + `isMaybeSingle` behavior → `cardinality: "maybe"` (note: `maybeSingle` is a client-side behavior in postgrest-js using the same Accept header for non-GET, and JSON array + client filtering for GET — the translator can't distinguish this from raw HTTP alone; closest mapping is `cardinality: "one"` for object+json Accept)

**HEAD requests:**

If HTTP method = HEAD → `$meta.head: true`

**From query params:**

| Param | AST $meta field |
|-------|-----------------|
| `columns` | `columns: ["col1", "col2"]` (from quoted CSV) |

---

### 3.5 resolveRpcParams

RPC-specific field resolution. Only runs when `route.isRpc === true`.

```ts
function resolveRpcParams(
  route: RouteResult,
  method: string,
  queryParams: QueryParamsResult,
  body: BodyResult
): {
  args?: object
  httpMethod: "GET" | "POST"
  paramsType?: "named" | "positional"
  inputType?: "json" | "text" | "binary" | "xml"
}
```

**POST RPC:**

- `args` comes from body
- `httpMethod: "POST"`
- Array body → `paramsType: "positional"`; object body → `paramsType: "named"`
- `inputType` from `Content-Type` header (json/text/xml/binary)

From `PostgrestClient.rpc()`:
```ts
// POST case
method = 'POST'
body = args
```

**GET RPC:**

- `httpMethod: "GET"`
- Args come from query params that are **not** operator-prefixed
- Filters come from query params that **are** operator-prefixed
- `paramsType: "named"` (GET always uses named params in query string)
- `inputType: "json"` (implicit)

From `PostgrestClient.rpc()` GET case:
```ts
Object.entries(args)
  .filter(([_, value]) => value !== undefined)
  .map(([name, value]) => [name, Array.isArray(value) ? `{${value.join(',')}}` : `${value}`])
  .forEach(([name, value]) => {
    url.searchParams.append(name, value)
  })
```

**GET arg vs filter disambiguation heuristic:**

When receiving a raw GET RPC request, query params are ambiguous:

```
/rpc/search?term=hello&status=eq.active
```

- `term=hello` — no operator prefix → **arg** (value = `"hello"`)
- `status=eq.active` — has operator prefix `eq.` → **filter** (on result set)

Heuristic: if value matches `{operator}.{rest}` where operator ∈ known PostgREST operators → filter. Otherwise → arg.

```ts
const OPERATORS = new Set([
  'eq','neq','gt','gte','lt','lte','like','ilike','match','imatch',
  'is','isdistinct','in','cs','cd','sl','sr','nxl','nxr','adj','ov',
  'fts','plfts','phfts','wfts','not'
])

function isFilter(value: string): boolean {
  const dot = value.indexOf('.')
  if (dot === -1) return false
  const prefix = value.substring(0, dot)
  // Handle quantified: like(any), like(all)
  const base = prefix.replace(/\((any|all)\)$/, '')
  return OPERATORS.has(base)
}
```

**Edge case:** a plain arg value that looks like an operator (e.g. `?mode=eq.balanced`). The heuristic misclassifies this. Mitigation: function authors should avoid parameter names that could collide. This matches PostgREST's own behavior.

**`Prefer: params=single-object`:** This tells PostgREST to pass the entire JSON body as a single argument. Transport-only concern — doesn't change AST shape (args is still the parsed body object). Could optionally record in `$meta` for backend awareness.

---

### 3.6 resolveUpsertParams

Extracts upsert-specific fields. Only runs when `type === "upsert"`.

```ts
function resolveUpsertParams(
  queryParams: QueryParamsResult,
  headers: HeadersResult
): {
  onConflict?: string
  ignoreDuplicates: boolean
}
```

**Logic:**

- `on_conflict` query param → `onConflict` (comma-separated column names)
- `resolution=ignore-duplicates` Prefer token → `ignoreDuplicates: true`
- `resolution=merge-duplicates` Prefer token → `ignoreDuplicates: false` (default)

From `PostgrestQueryBuilder.upsert()`:
```ts
headers.append('Prefer', `resolution=${ignoreDuplicates ? 'ignore' : 'merge'}-duplicates`)
if (onConflict !== undefined) url.searchParams.set('on_conflict', onConflict)
```

---

## 4. Select Grammar Reference

### Full EBNF

```ebnf
(* Top-level *)
query          = node { "," node } ;
node           = star | spread | renamed_field | field ;

(* Node types *)
star           = "*" ;
spread         = "..." field_with_embed ;
renamed_field  = identifier ":" field ;
field          = field_with_embed | non_embed_field ;

(* Embedded resource field *)
field_with_embed = identifier [ modifiers ] "(" [ query ] ")" ;
modifiers        = "!" modifier_body ;
modifier_body    = "inner"
                 | "left"
                 | identifier                          (* hint *)
                 | identifier "!" ( "inner" | "left" ) (* hint + join type *)
                 ;

(* Non-embedded field *)
non_embed_field  = identifier [ json_path ] [ typecast ] [ aggregation ] [ typecast ] ;
json_path        = arrow identifier { arrow identifier } ;
arrow            = "->" | "->>" ;
typecast         = "::" identifier ;
aggregation      = "." agg_function "()" ;
agg_function     = "count" | "sum" | "avg" | "min" | "max" ;

(* Special case: standalone count *)
count_field      = "count" [ "()" ] [ typecast ] ;

(* Identifiers *)
identifier       = unquoted_id | quoted_id ;
unquoted_id      = letter { letter } ;
quoted_id        = '"' { any_non_quote } '"' ;
letter           = "a"-"z" | "A"-"Z" | "0"-"9" | "_" ;
```

### Recursive descent walkthrough

```
Input: "id,desc:description,categories!inner(id,name),price.avg()::int"

1. parseQuery()
   ├─ parseNode() → "id"                                    [simple field]
   ├─ ","
   ├─ parseNode()
   │  ├─ parseIdentifier() → "desc"
   │  ├─ ":" → alias = "desc"
   │  └─ parseField() → { column: "description" }           [renamed field]
   ├─ ","
   ├─ parseNode()
   │  ├─ parseIdentifier() → "categories"
   │  ├─ "!" → parseModifiers()
   │  │  └─ "inner" → innerJoin = true
   │  └─ "(" → parseQuery() recursively
   │     ├─ "id"
   │     ├─ ","
   │     ├─ "name"
   │     └─ ")" → children = ["id", "name"]                 [embedded resource]
   ├─ ","
   └─ parseNode()
      ├─ parseIdentifier() → "price"
      ├─ "." → parseAggregation()
      │  └─ "avg()" → aggregate = "avg"
      └─ "::" → parseCast()
         └─ "int" → cast = "int"                            [aggregated field]
```

**AST output:**

```json
{
  "select": [
    "id",
    { "desc": { "column": "description" } },
    { "categories": {
        "select": ["id", "name"]
    }},
    { "price": { "column": "price", "aggregate": "avg", "cast": "int" } }
  ],
  "join": {
    "categories": { "type": "inner" }
  }
}
```

### JSON path parse example

```
Input: "metadata->theme->>color"

1. parseIdentifier() → "metadata"
2. "->" → parseJsonAccessor()
   ├─ ">" → single arrow → json type
   │  └─ parseIdentifier() → "theme"
   └─ "->>" → double arrow → text type
      └─ parseIdentifier() → "color"

→ { "color": { "column": "metadata", "path": "$.theme.color" } }
```

Arrow rules:
- `->` = JSON traversal (returns JSON)
- `->>` = JSON traversal (returns text) — only valid as final accessor
- PostgREST represents this as property chain; AST uses JSONPath syntax

---

## 5. Operator Reference Table

### PostgREST → AST operator mapping

| PostgREST operator | AST operator | Value format | Notes |
|-------------------|--------------|--------------|-------|
| `eq` | `$eq` | scalar | |
| `neq` | `$neq` | scalar | |
| `gt` | `$gt` | scalar | |
| `gte` | `$gte` | scalar | |
| `lt` | `$lt` | scalar | |
| `lte` | `$lte` | scalar | |
| `like` | `$like` | string (pattern) | |
| `ilike` | `$ilike` | string (pattern) | |
| `like(any)` | `$likeAny` | `{pat1,pat2}` → array | |
| `like(all)` | `$likeAll` | `{pat1,pat2}` → array | |
| `ilike(any)` | `$ilikeAny` | `{pat1,pat2}` → array | |
| `ilike(all)` | `$ilikeAll` | `{pat1,pat2}` → array | |
| `match` | `$regex` | string (pattern) | postgrest-js: `regexMatch()` |
| `imatch` | `$iregex` | string (pattern) | postgrest-js: `regexIMatch()` |
| `is` | `$is` | `true`/`false`/`null` | |
| `isdistinct` | `$isDistinct` | scalar | |
| `in` | `$in` | `(v1,v2,v3)` → array | |
| `not.in` | `$notIn` | `(v1,v2,v3)` → array | shorthand for NOT IN |
| `cs` | `$contains` | `{v1,v2}` or JSON string | array/json/range |
| `cd` | `$containedBy` | `{v1,v2}` or JSON string | array/json/range |
| `ov` | `$overlaps` | `{v1,v2}` or range string | |
| `sl` | `$rangeLt` | range string | `<<` |
| `sr` | `$rangeGt` | range string | `>>` |
| `nxl` | `$rangeGte` | range string | `&>` — not extends left |
| `nxr` | `$rangeLte` | range string | `&<` — not extends right |
| `adj` | `$rangeAdjacent` | range string | `-\|-` |
| `fts` | `$textSearch` | query string | type: (basic) |
| `plfts` | `$textSearch` | query string | type: `"plain"` |
| `phfts` | `$textSearch` | query string | type: `"phrase"` |
| `wfts` | `$textSearch` | query string | type: `"websearch"` |

### Quantified comparison operators

Formed as `{base}(any)` or `{base}(all)` in PostgREST:

| PostgREST | AST |
|-----------|-----|
| `eq(any)` | `$eqAny` |
| `eq(all)` | `$eqAll` |
| `gt(any)` | `$gtAny` |
| `gt(all)` | `$gtAll` |
| `gte(any)` | `$gteAny` |
| `gte(all)` | `$gteAll` |
| `lt(any)` | `$ltAny` |
| `lt(all)` | `$ltAll` |
| `lte(any)` | `$lteAny` |
| `lte(all)` | `$lteAll` |
| `like(any)` | `$likeAny` |
| `like(all)` | `$likeAll` |
| `ilike(any)` | `$ilikeAny` |
| `ilike(all)` | `$ilikeAll` |
| `match(any)` | `$regexAny` |
| `match(all)` | `$regexAll` |
| `imatch(any)` | `$iregexAny` |
| `imatch(all)` | `$iregexAll` |

Value format for quantified operators: `{val1,val2,val3}` → parse to array.

### Negation

Prefix `not.` before any operator:

```
status=not.eq.active         → { "status": { "$not": { "$eq": "active" } } }
id=not.in.(1,2,3)            → { "id": { "$notIn": [1, 2, 3] } }
name=not.like.*test*         → { "name": { "$not": { "$like": "*test*" } } }
```

**Special case:** `not.in` maps to the dedicated `$notIn` AST operator (shorthand) rather than `$not: { $in: [...] }`. Both are semantically equivalent; `$notIn` is preferred.

From `PostgrestFilterBuilder.not()`:
```ts
this.url.searchParams.append(column, `not.${operator}.${value}`)
```

And for `notIn`:
```ts
this.url.searchParams.append(column, `not.in.(${cleanedValues})`)
```

### Full-text search with config

```
description=fts(english).phone & flagship
→ { "description": { "$textSearch": { "query": "phone & flagship", "type": "plain", "config": "english" } } }
```

Wait — correction. `fts` = basic (no type prefix), `plfts` = plain, `phfts` = phrase, `wfts` = websearch.

From `PostgrestFilterBuilder.textSearch()`:
```ts
let typePart = ''
if (type === 'plain')     typePart = 'pl'
else if (type === 'phrase')    typePart = 'ph'
else if (type === 'websearch') typePart = 'w'
const configPart = config === undefined ? '' : `(${config})`
this.url.searchParams.append(column, `${typePart}fts${configPart}.${query}`)
```

So parsing: `{prefix}fts{(config)}.{query}`
- prefix = "" → basic, "pl" → plain, "ph" → phrase, "w" → websearch
- config is optional, in parentheses
- query is everything after the dot

```
description=fts.phone           → $textSearch: { query: "phone" }
description=plfts(english).phone → $textSearch: { query: "phone", type: "plain", config: "english" }
description=wfts.phone          → $textSearch: { query: "phone", type: "websearch" }
```

### Value parsing rules

| Value format | Parse rule | Example |
|-------------|------------|---------|
| Scalar | String by default; attempt numeric parse for comparison ops | `gt.100` → `100` |
| `(v1,v2,v3)` | Split on `,`, respecting quoted strings | `in.(1,2,"a,b")` → `[1, 2, "a,b"]` |
| `{v1,v2,v3}` | Array literal — split on `,` | `cs.{1,2,3}` → `[1, 2, 3]` |
| `true`/`false`/`null` | Boolean/null literals for `is` operator | `is.true` → `true` |
| JSON string | Parse as JSON for `cs`/`cd` | `cs.{"key":"val"}` → `{"key":"val"}` |
| Range string | Pass through as string | `sr.[1,5]` → `"[1,5]"` |

**Quoted strings in `in.()`:**

PostgREST reserves `,()` characters. Strings containing them must be double-quoted:

```
in.(hello,"world,2",foo)  → ["hello", "world,2", "foo"]
```

From `PostgrestFilterBuilder.in()`:
```ts
if (typeof s === 'string' && PostgrestReservedCharsRegexp.test(s)) return `"${s}"`
```

---

## 6. Composability & Extension

### createTranslator API

```ts
type TranslatorConfig = {
  parseRoute?:        (req: Request) => RouteResult
  parseHeaders?:      (req: Request) => HeadersResult
  parseSelect?:       (req: Request) => SelectResult
  parseBody?:         (req: Request) => Promise<BodyResult>
  parseQueryParams?:  (req: Request) => QueryParamsResult
  resolveType?:       (route, method, headers) => ASTType
  resolveFilters?:    (queryParams, embeddedAliases) => FiltersResult
  resolveTransforms?: (queryParams, embeddedAliases) => TransformsResult
  resolveMeta?:       (headers, queryParams) => object
  resolveRpcParams?:  (route, method, queryParams, body) => RpcResult
  resolveUpsertParams?: (queryParams, headers) => UpsertResult
  basePath?:          string  // default: "/rest/v1"
}

function createTranslator(config?: TranslatorConfig): {
  translate: (req: Request) => Promise<AST>
}
```

Each slot has a clear input→output contract. Override any parser/resolver without affecting others.

### Swap example — custom route parser

```ts
const translator = createTranslator({
  parseRoute(req) {
    // Custom routing: /api/v2/{table} instead of /rest/v1/{table}
    const path = new URL(req.url).pathname.replace(/^\/api\/v2\//, '')
    const segments = path.split('/')
    if (segments[0] === 'rpc') {
      return { function: segments[1], isRpc: true }
    }
    return { from: segments[0], isRpc: false }
  }
})
```

### Extension example — add custom operator

```ts
const translator = createTranslator({
  resolveFilters(queryParams, embeddedAliases) {
    const base = defaultResolveFilters(queryParams, embeddedAliases)
    // Add custom operator support
    // e.g., `column=geo_within.{lat},{lng},{radius}`
    return base
  }
})
```

### Middleware pattern

Pre/post hooks for request preprocessing or AST postprocessing:

```ts
const translator = createTranslator()

async function translateWithHooks(req: Request): Promise<AST> {
  // Pre-hook: normalize headers, add defaults
  const normalizedReq = preprocess(req)

  const ast = await translator.translate(normalizedReq)

  // Post-hook: enforce RLS, inject schema, add audit fields
  return postprocess(ast)
}
```

---

## 7. Edge Cases

### RPC GET arg vs filter disambiguation

See §3.5. The heuristic is operator-prefix detection. Failure mode: arg values that look like operator expressions (e.g. `?mode=eq.balanced`). Recommendation: function parameter names should avoid this collision. PostgREST itself has the same limitation.

### Column name vs join alias collision

When `where` key matches both a table column and a join alias, the join alias takes precedence:

```json
// Table "posts" has column "author" (text) and join alias "author" (→ users table)
{ "where": { "author": { "$eq": null } } }
// → anti-join on author relationship, NOT column filter
```

Resolution: rename the join alias to avoid collision.

### Quoted identifiers in select and filter params

Select: `"my column"` → double-quoted in select string, parsed by `ParseQuotedLetters`.

Filters: column names with special chars must be URL-encoded. PostgREST accepts them.

### `in.(...)` value parsing with quoted strings

```
in.(foo,"bar,baz",qux)
```

Parser must handle nested quotes within parentheses:
1. Strip outer `()`
2. Split on `,` but not within `"..."`
3. Unquote each value: `"bar,baz"` → `bar,baz`

### Multiple values for same query param key

```
?price=gte.100&price=lte.500
```

Both values apply as AND semantics. Result:
```json
{ "price": { "$gte": 100, "$lte": 500 } }
```

Same key, same operator → last value wins (PostgREST behavior).

### Empty body, empty select, `select=*`

- **Empty body:** POST/PATCH with empty body → `values: {}` (empty object). For insert, PostgREST inserts a row with all defaults.
- **Empty select:** no `select` param → omit `select` from AST (backend returns all columns).
- **`select=*`:** explicit wildcard → `select: ["*"]`. Functionally equivalent to omitted select, but preserves the explicit intent. (See Open Questions §9.)

### Embedded resource with empty subselect

```
select=id,categories()
```

Empty parens → embed with all columns: `{ "categories": { "select": ["*"] } }`

### Type casting in filters

PostgREST supports `column::type=op.value` in URL params. The translator should preserve the cast in the where entry via the `path` or a dedicated `cast` field on the filter. This is rare in practice.

---

## 8. Error Handling

### Error types

```ts
type TranslationError = {
  type: "parse_error" | "validation_error" | "unsupported_feature"
  message: string
  source: "route" | "headers" | "select" | "body" | "query_params" | "resolver"
  position?: { offset: number; line: number; column: number }  // for select parse errors
  param?: string   // the query param or header that caused the error
}
```

### Parse errors with position tracking

The select parser should track character position for error messages:

```
select=id,categories(!inner(id,name)
                                    ^ Expected ")" at position 36
```

### Strict vs lenient modes

Matches PostgREST's `Prefer: handling=strict|lenient`:

- **Strict** (`handling=strict`): unknown query params, invalid operators, malformed values → throw `TranslationError`
- **Lenient** (default): unknown query params are ignored, best-effort parsing, invalid operators → `$filter` escape hatch

When `$meta.handling === "strict"`, the translator should fail on any unrecognized input rather than silently dropping it.

### Common error scenarios

| Scenario | Error type | Behavior |
|----------|-----------|----------|
| Malformed select syntax | `parse_error` | Always error (can't guess intent) |
| Unknown operator in filter | `validation_error` | Strict: error. Lenient: use `$filter` |
| Invalid `in.(...)` syntax | `parse_error` | Always error |
| Non-numeric limit/offset | `validation_error` | Strict: error. Lenient: ignore |
| Unmatched parens in `or=()` | `parse_error` | Always error |
| Unknown Prefer token | `validation_error` | Strict: error. Lenient: ignore |
| Body parse failure | `parse_error` | Always error (invalid JSON) |

---

## 9. Open Questions

1. **`select=*` → literal `["*"]` or omit `select`?** Recommendation: `["*"]` to preserve explicit intent. Omitted `select` means "no select param was sent." Functionally equivalent but semantically distinct.

2. **`put` type — AST-only?** Yes. The translator never produces `type: "put"`. It exists for direct AST construction by backends that want full-row replacement semantics without going through PostgREST HTTP conventions.

3. **RPC GET arg/filter disambiguation — operator-prefix heuristic sufficient?** For practical purposes, yes. Matches PostgREST's own behavior. Edge case: args whose values happen to start with an operator name + dot. Mitigation: function authors avoid collisions. Alternative: schema-aware disambiguation (requires function signature metadata).

4. **`Prefer: params=single-object`** — transport-only. Doesn't change AST shape. Could optionally store in `$meta.paramsType: "single-object"` for backends that need to reconstruct the Prefer header.

5. **TypeScript types inline or separate?** The types shown in this document are illustrative. A real implementation should define them in a shared types module and reference from here.

---

## 10. End-to-End Examples

### Example 1: SELECT with embeds, filters, ordering

**Request:**
```http
GET /rest/v1/products?select=id,name,price,categories!inner(id,name),reviews(rating,comment)&status=eq.active&price=gt.100&price=lt.500&categories.active=eq.true&order=price.asc.nullsfirst,name.desc&reviews.order=created_at.desc&limit=50&offset=0
Accept: application/json
Accept-Profile: public
Prefer: count=exact
```

**Phase 1:**

`parseRoute` → `{ from: "products", isRpc: false }`

`parseHeaders` → `{ schema: "public", preferTokens: [{ key: "count", value: "exact" }], accept: "application/json" }`

`parseSelect` →
```json
{
  "select": [
    "id", "name", "price",
    { "categories": { "select": ["id", "name"] } },
    { "reviews": { "select": ["rating", "comment"] } }
  ],
  "join": {
    "categories": { "type": "inner" },
    "reviews": {}
  },
  "embeddedAliases": ["categories", "reviews"]
}
```

`parseBody` → `{}` (GET, no body)

`parseQueryParams` →
```
select → ["id,name,price,categories!inner(id,name),reviews(rating,comment)"]
status → ["eq.active"]
price  → ["gt.100", "lt.500"]
categories.active → ["eq.true"]
order  → ["price.asc.nullsfirst,name.desc"]
reviews.order → ["created_at.desc"]
limit  → ["50"]
offset → ["0"]
```

**Phase 2:**

`resolveType` → `"query"` (GET + table)

`resolveFilters` →
```json
{
  "where": {
    "status": { "$eq": "active" },
    "price": { "$gt": 100, "$lt": 500 }
  },
  "embeddedWheres": {
    "categories": { "active": { "$eq": true } }
  }
}
```

`resolveTransforms` →
```json
{
  "order": [
    { "column": "price", "direction": "asc", "nullsFirst": true },
    { "column": "name", "direction": "desc" }
  ],
  "limit": 50,
  "offset": 0,
  "embeddedTransforms": {
    "reviews": {
      "order": [{ "column": "created_at", "direction": "desc" }]
    }
  }
}
```

`resolveMeta` → `{ "count": "exact", "cardinality": "many" }`

**Final AST:**

```json
{
  "type": "query",
  "from": "products",
  "schema": "public",
  "join": {
    "categories": { "type": "inner" },
    "reviews": {}
  },
  "select": [
    "id", "name", "price",
    { "categories": {
        "select": ["id", "name"],
        "where": { "active": { "$eq": true } }
    }},
    { "reviews": {
        "select": ["rating", "comment"],
        "order": [{ "column": "created_at", "direction": "desc" }]
    }}
  ],
  "where": {
    "status": { "$eq": "active" },
    "price": { "$gt": 100, "$lt": 500 }
  },
  "order": [
    { "column": "price", "direction": "asc", "nullsFirst": true },
    { "column": "name", "direction": "desc" }
  ],
  "limit": 50,
  "offset": 0,
  "$meta": {
    "count": "exact"
  }
}
```

---

### Example 2: RPC GET with mixed args + filters

**Request:**
```http
GET /rest/v1/rpc/search_products?term=phone&category=electronics&min_rating=gte.4&status=eq.available&select=id,name,score&order=score.desc&limit=20
Accept-Profile: public
```

**Phase 1:**

`parseRoute` → `{ function: "search_products", isRpc: true }`

`parseQueryParams` →
```
term → ["phone"]
category → ["electronics"]
min_rating → ["gte.4"]
status → ["eq.available"]
select → ["id,name,score"]
order → ["score.desc"]
limit → ["20"]
```

**Phase 2:**

`resolveType` → `"rpc"`

`resolveRpcParams`:
- `term=phone` — no operator prefix → **arg**
- `category=electronics` — no operator prefix → **arg**
- `min_rating=gte.4` — `gte` is known operator → **filter**
- `status=eq.available` — `eq` is known operator → **filter**

```json
{
  "args": { "term": "phone", "category": "electronics" },
  "httpMethod": "GET",
  "paramsType": "named",
  "inputType": "json"
}
```

`resolveFilters` (from operator-prefixed params):
```json
{ "where": { "min_rating": { "$gte": 4 }, "status": { "$eq": "available" } } }
```

**Final AST:**

```json
{
  "type": "rpc",
  "function": "search_products",
  "schema": "public",
  "args": { "term": "phone", "category": "electronics" },
  "httpMethod": "GET",
  "paramsType": "named",
  "inputType": "json",
  "select": ["id", "name", "score"],
  "where": {
    "min_rating": { "$gte": 4 },
    "status": { "$eq": "available" }
  },
  "order": [{ "column": "score", "direction": "desc" }],
  "limit": 20
}
```

---

### Example 3: Upsert with Prefer headers

**Request:**
```http
POST /rest/v1/inventory?on_conflict=product_id&select=product_id,quantity,updated_at&columns="product_id","quantity"
Content-Type: application/json
Content-Profile: public
Prefer: resolution=merge-duplicates, count=exact, missing=default, return=representation

[
  { "product_id": 1, "quantity": 50 },
  { "product_id": 2, "quantity": 30 }
]
```

**Phase 1:**

`parseRoute` → `{ from: "inventory", isRpc: false }`

`parseHeaders` →
```json
{
  "schema": "public",
  "preferTokens": [
    { "key": "resolution", "value": "merge-duplicates" },
    { "key": "count", "value": "exact" },
    { "key": "missing", "value": "default" },
    { "key": "return", "value": "representation" }
  ],
  "accept": "application/json"
}
```

`parseBody` →
```json
{
  "values": [
    { "product_id": 1, "quantity": 50 },
    { "product_id": 2, "quantity": 30 }
  ]
}
```

**Phase 2:**

`resolveType` → `"upsert"` (POST + `resolution=merge-duplicates`)

`resolveUpsertParams` → `{ "onConflict": "product_id", "ignoreDuplicates": false }`

`resolveMeta` → `{ "count": "exact", "missing": "default", "columns": ["product_id", "quantity"] }`

**Final AST:**

```json
{
  "type": "upsert",
  "from": "inventory",
  "schema": "public",
  "values": [
    { "product_id": 1, "quantity": 50 },
    { "product_id": 2, "quantity": 30 }
  ],
  "onConflict": "product_id",
  "ignoreDuplicates": false,
  "select": ["product_id", "quantity", "updated_at"],
  "$meta": {
    "count": "exact",
    "missing": "default",
    "columns": ["product_id", "quantity"]
  }
}
```
