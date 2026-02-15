# Database API Object Representation AST

Unified JSON format AST covering all 66+ Database API methods.

---

## Core Structure

```json
{
  "type": "query",
  "from": "table",
  "schema": "public",
  "join": {},
  "select": [],
  "where": {},
  "order": [],
  "limit": 10,
  "offset": 0,
  "group": [],
  "$meta": {}
}
```

**Operation types:** `query` (default), `insert`, `update`, `delete`, `upsert`, `put`, `rpc`

---

## 1. Query (SELECT)

```json
{
  "from": "products",
  "join": {
    "categories": {},
    "reviews": { "type": "inner" }
  },
  "select": [
    "id",
    "name",
    "price",
    { "desc": { "column": "description" } },
    { "color": { "column": "metadata", "path": "$.theme.color" } },
    { "first_tag": { "column": "tags", "path": "$[0]" } },
    { "categories": {
        "select": ["id", "name"],
        "where": { "active": { "$is": true } },
        "limit": 5
    }},
    { "reviews": {
        "select": ["rating", "comment"],
        "order": [{ "column": "created_at", "direction": "desc" }]
    }}
  ],
  "where": {
    "status": { "$eq": "active" },
    "price": { "$gt": 100, "$lt": 500 },
    "deleted_at": { "$is": null }
  },
  "order": [
    { "column": "price", "direction": "asc" },
    { "column": "name", "direction": "desc", "nullsFirst": true }
  ],
  "limit": 50,
  "offset": 0,
  "$meta": {
    "cardinality": "many",
    "count": "exact"
  }
}
```

---

## 2. Insert

```json
{
  "type": "insert",
  "from": "products",
  "values": [
    { "name": "iPhone 15", "price": 999 },
    { "name": "Galaxy S24", "price": 899 }
  ],
  "select": ["id", "name", "created_at"],
  "$meta": {
    "missing": "default",
    "columns": ["name", "price"],
    "count": "exact"
  }
}
```

---

## 3. Update

```json
{
  "type": "update",
  "from": "products",
  "values": {
    "price": 799,
    "updated_at": "2026-02-02T00:00:00Z"
  },
  "where": {
    "id": { "$eq": 123 }
  },
  "select": ["id", "price", "updated_at"],
  "$meta": {
    "maxAffected": 1
  }
}
```

---

## 4. Delete

```json
{
  "type": "delete",
  "from": "products",
  "where": {
    "id": { "$in": [1, 2, 3] }
  },
  "select": ["id", "name"],
  "$meta": {
    "maxAffected": 10
  }
}
```

---

## 5. Upsert

```json
{
  "type": "upsert",
  "from": "inventory",
  "values": [
    { "product_id": 1, "quantity": 50 },
    { "product_id": 2, "quantity": 30 }
  ],
  "onConflict": "product_id",
  "ignoreDuplicates": false,
  "select": ["product_id", "quantity", "updated_at"]
}
```

---

## 6. Put

Full row replacement. Unlike upsert (merge), put replaces the entire row — missing columns become null/default.

```json
{
  "type": "put",
  "from": "users",
  "values": { "id": 1, "name": "John", "email": "john@example.com" },
  "select": ["id", "name", "email"]
}
```

SQL: `INSERT ... ON CONFLICT(pk) DO UPDATE SET col1=excluded.col1, col2=excluded.col2, ...` (all columns).

---

## 7. RPC

```json
{
  "type": "rpc",
  "function": "calculate_discount",
  "args": {
    "product_id": 123,
    "discount_percent": 15
  },
  "httpMethod": "GET",
  "paramsType": "named",
  "inputType": "json",
  "select": ["final_price", "tax_amount"],
  "where": {
    "status": { "$eq": "success" }
  },
  "order": [{ "column": "final_price", "direction": "asc" }],
  "limit": 10
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `httpMethod` | `"GET"` \| `"POST"` | `"POST"` | GET = read-only (stable/immutable), POST = transactional |
| `paramsType` | `"named"` \| `"positional"` | `"named"` | Named = `{ "a": 1 }`, positional = `[1, 2]` |
| `inputType` | `"json"` \| `"text"` \| `"binary"` \| `"xml"` | `"json"` | Request body content type |

---

## Operators

### Comparison

| Operator | SQL | Notes |
|----------|-----|-------|
| `$eq` | `=` / `IS NULL` | `$eq: null` → `IS NULL` (shorthand) |
| `$neq` | `<>` / `IS NOT NULL` | `$neq: null` → `IS NOT NULL` (shorthand) |
| `$gt` | `>` | |
| `$gte` | `>=` | |
| `$lt` | `<` | |
| `$lte` | `<=` | |
| `$in` | `IN (...)` | |
| `$notIn` | `NOT IN (...)` | |
| `$is` | `IS` | Values: `true`, `false`, `"unknown"`, `null` |
| `$isDistinct` | `IS DISTINCT FROM` | NULL-safe inequality |

**Quantified variants** — each takes array value:

```
$eqAny, $eqAll, $gtAny, $gtAll, $gteAny, $gteAll,
$ltAny, $ltAll, $lteAny, $lteAll
```

### Pattern Matching

| Operator | SQL |
|----------|-----|
| `$like` | `LIKE` |
| `$ilike` | `ILIKE` |
| `$likeAny` | `LIKE ANY(...)` |
| `$likeAll` | `LIKE ALL(...)` |
| `$ilikeAny` | `ILIKE ANY(...)` |
| `$ilikeAll` | `ILIKE ALL(...)` |
| `$regex` | `~` |
| `$iregex` | `~*` |
| `$regexAny` | `~ ANY(...)` |
| `$regexAll` | `~ ALL(...)` |
| `$iregexAny` | `~* ANY(...)` |
| `$iregexAll` | `~* ALL(...)` |

### Array/JSON

| Operator | SQL | Notes |
|----------|-----|-------|
| `$contains` | `@>` | Array/JSON contains |
| `$containedBy` | `<@` | Contained by |
| `$overlaps` | `&&` | Has common elements |

### Range (PostgreSQL only)

| Operator | SQL |
|----------|-----|
| `$rangeGt` | `>>` |
| `$rangeGte` | `&>` |
| `$rangeLt` | `<<` |
| `$rangeLte` | `&<` |
| `$rangeAdjacent` | `-|-` |

### Full-Text Search

```json
{
  "where": {
    "description": {
      "$textSearch": {
        "query": "phone & flagship",
        "type": "websearch",
        "config": "english"
      }
    }
  }
}
```

Types: `plain`, `phrase`, `websearch`, or omit for basic.

### Logical

**`$not` (wrapper)** — negates any operator or compound:

```json
{
  "where": { "status": { "$not": { "$eq": "active" } } }
}
```

```json
{
  "where": { "name": { "$not": { "$likeAny": ["%test%", "%demo%"] } } }
}
```

```json
{
  "where": { "$not": { "$and": [{ "a": { "$gte": 0 } }, { "a": { "$lte": 100 } }] } }
}
```

**`$or` / `$and`:**

```json
{
  "where": {
    "$or": [
      { "category": { "$eq": "phones" } },
      { "featured": { "$is": true } }
    ]
  }
}
```

### Match (shorthand for multiple $eq)

```json
{
  "where": {
    "$match": {
      "category": "electronics",
      "status": "active"
    }
  }
}
```

### Raw Filter (escape hatch)

```json
{
  "where": {
    "$filter": {
      "column": "custom_col",
      "operator": "custom_op",
      "value": "val"
    }
  }
}
```

---

## Select Formats

### Column entries

```json
"select": [
  "id",
  { "desc": { "column": "description" } },
  { "salary_text": { "column": "salary", "cast": "text" } },
  { "salary": { "cast": "text" } },
  { "color": { "column": "metadata", "path": "$.theme.color" } },
  { "total": { "column": "price", "aggregate": "sum" } },
  { "row_count": { "aggregate": "count" } },
  { "avg_price": { "column": "price", "aggregate": "avg", "cast": "int" } },
  { "total_amt": { "column": "amount", "preCast": "numeric", "aggregate": "sum" } }
]
```

| Field | Type | Description |
|-------|------|-------------|
| key | string | Output alias (required) |
| `column` | string | Source column. Omit if same as alias |
| `cast` | string | Type cast (post-aggregate when with `aggregate`) |
| `preCast` | string | Type cast before aggregate (rare) |
| `aggregate` | `"sum"` \| `"avg"` \| `"max"` \| `"min"` \| `"count"` | Aggregate function |
| `path` | string | JSONPath for JSON/array access |

**Cast ordering:** `preCast` → column → `aggregate` → `cast`

- `{ "column": "amount", "preCast": "numeric", "aggregate": "sum" }` → `SUM(amount::numeric)`
- `{ "column": "price", "aggregate": "avg", "cast": "int" }` → `AVG(price)::int`

**Aggregates:**

- With `column`: `COUNT(email)`, `SUM(price)`, etc.
- Without `column`: `COUNT(*)` (row count)
- Top-level `group` required when mixing aggregates + plain columns

### Embed entries

Embed entries pull data from related tables via JSON aggregate subqueries — not via SQL JOINs. The `join` field only establishes relationships for filtering.

```json
"select": [
  { "actors": {
      "select": ["name", "age"],
      "where": { "active": { "$is": true } },
      "order": [{ "column": "name" }],
      "limit": 10,
      "offset": 0
  }},
  { "director": {
      "spread": true,
      "select": [{ "director_name": { "column": "first_name" } }]
  }}
]
```

**SQL translation:**

- PG: `(SELECT json_agg(row_to_json(t.*)) FROM actors t WHERE t.film_id = films.id AND t.active IS TRUE ORDER BY t.name LIMIT 10) AS actors`
- SQLite: `(SELECT json_group_array(json_object('name', t.name, 'age', t.age)) FROM actors t WHERE t.film_id = films.id AND t.active = 1 ORDER BY t.name LIMIT 10) AS actors`

| Field | Type | Description |
|-------|------|-------------|
| key | string | Join alias (must match `join` entry or implies relationship) |
| `select` | array | Columns from related table |
| `where` | object | Filter on embedded resource (within subquery) |
| `order` | array | Order within embedded resource (within subquery) |
| `limit` | number | Limit embedded rows (within subquery) |
| `offset` | number | Offset embedded rows (within subquery) |
| `spread` | boolean | Flatten fields into parent (to-one only) |
| `join` | object | Nested join definitions (for deeper embeds) |

**Parser rule:** if a select entry's value object contains `select`, it's an embed entry. Otherwise it's a column entry.

### JSONPath

- `$.key` — object property
- `$.nested.key` — nested property
- `$[0]` — array index
- `$.items[0].name` — combined

---

## Join Format

```json
"join": {
  "actors": {},
  "director": { "from": "directors", "type": "inner", "hint": "director_id" }
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| key | string | — | Alias for this join |
| `from` | string | alias | Table name. Omit when alias == table name |
| `type` | `"left"` \| `"inner"` | `"left"` | Join type |
| `hint` | string | auto | FK column for disambiguation |

**Design principle: joins are always for filtering.** They establish relationships for `where` conditions, existence checks, and anti-joins. They never add data to the response. Data from related tables comes exclusively from embed entries in `select`, which translate to JSON aggregate subqueries (`json_agg` in PG, `json_group_array` in SQLite).

**Rules:**

- **Implicit join:** embed in `select` without `join` entry → auto left join (for the subquery)
- **Inner join:** `type: "inner"` → `INNER JOIN` — excludes parent rows without matches
- **Left join:** default — relationship available for `where` dot notation, doesn't filter parent rows on its own
- **Multiple joins to same table:** different aliases, each with `from`
- **Minimal:** `"actors": {}` — left join, auto-detect FK

---

## Order Format

```json
"order": [
  { "column": "price", "direction": "asc" },
  { "column": "name", "direction": "desc", "nullsFirst": true },
  { "column": "director.last_name", "direction": "desc" }
]
```

| Field | Type | Default |
|-------|------|---------|
| `column` | string | required |
| `direction` | `"asc"` \| `"desc"` | `"asc"` |
| `nullsFirst` | boolean | `false` |

Dot notation (`director.last_name`) references joined table columns. Requires matching `join` entry. To-one joins only.

---

## $meta Fields

| Field | Type | Operations | Description |
|-------|------|------------|-------------|
| `cardinality` | string | query/rpc | `"one"` (error if ≠1), `"maybe"` (null if 0, error if >1), `"many"` (default) |
| `count` | string | all | `"exact"`, `"planned"`, `"estimated"` |
| `head` | boolean | query | Headers only, no body |
| `maxAffected` | number | update/delete | Limit affected rows |
| `rollback` | boolean | all | Return data but rollback transaction |
| `missing` | string | insert/upsert | `"null"` (default) or `"default"` — handling for missing columns |
| `handling` | string | all | `"strict"` or `"lenient"` — invalid param behavior |
| `timezone` | string | all | IANA timezone (e.g., `"America/Los_Angeles"`) |
| `columns` | array | insert/upsert | Restrict accepted payload columns |
| `stripNulls` | boolean | all | Strip null keys from response |
| `explain` | object | query | Query plan options |
| `headers` | object | all | Custom HTTP headers (transport) |

### Explain options

```json
{
  "$meta": {
    "explain": {
      "analyze": true,
      "verbose": false,
      "settings": true,
      "buffers": true,
      "wal": false
    }
  }
}
```

---

## Root Fields Summary

| Field | Type | Operations | Description |
|-------|------|------------|-------------|
| `type` | string | all | `query`/`insert`/`update`/`delete`/`upsert`/`put`/`rpc` |
| `from` | string | all except rpc | Table or view |
| `function` | string | rpc | Function name |
| `schema` | string | all | Schema name |
| `join` | object | all except rpc | Join definitions (filtering only) |
| `select` | array | all | Columns/embeds to return (embeds → JSON aggregate subqueries) |
| `where` | object | query/update/delete/rpc | Filters |
| `values` | object/array | insert/update/upsert/put | Data |
| `args` | object/array | rpc | Function arguments |
| `order` | array | query/rpc | Sort |
| `limit` | number | query/rpc | Row limit |
| `offset` | number | query/rpc | Row offset |
| `group` | array | query | Group-by columns for aggregates |
| `onConflict` | string | upsert | Conflict column(s) |
| `ignoreDuplicates` | boolean | upsert | Skip vs merge on conflict |
| `httpMethod` | string | rpc | `"GET"` / `"POST"` |
| `paramsType` | string | rpc | `"named"` / `"positional"` |
| `inputType` | string | rpc | `"json"` / `"text"` / `"binary"` / `"xml"` |
| `$meta` | object | all | Metadata/preferences |

---

## Examples

### Simple query

```json
{
  "from": "users",
  "select": ["id", "email"],
  "where": { "status": { "$eq": "active" } },
  "limit": 10
}
```

### Query with joins

```json
{
  "from": "posts",
  "join": {
    "author": { "type": "inner" }
  },
  "select": [
    "id", "title",
    { "author": { "select": ["name", "avatar"] } }
  ],
  "where": {
    "published": { "$is": true },
    "$or": [
      { "featured": { "$is": true } },
      { "views": { "$gt": 1000 } }
    ]
  }
}
```

### Single row

```json
{
  "from": "users",
  "select": ["id", "email", "profile"],
  "where": { "id": { "$eq": 123 } },
  "$meta": { "cardinality": "one" }
}
```

### Aggregates

```json
{
  "from": "orders",
  "select": [
    "category",
    { "total_revenue": { "column": "amount", "aggregate": "sum" } },
    { "avg_order": { "column": "amount", "aggregate": "avg", "cast": "int" } },
    { "order_count": { "aggregate": "count" } }
  ],
  "group": ["category"],
  "order": [{ "column": "category" }]
}
```

### Complex filters

```json
{
  "from": "products",
  "select": ["id", "name", "price"],
  "where": {
    "name": { "$ilike": "%phone%" },
    "price": { "$gte": 100, "$lte": 500 },
    "tags": { "$contains": ["featured"] },
    "deleted_at": { "$is": null },
    "$or": [
      { "category": { "$eq": "electronics" } },
      { "featured": { "$is": true } }
    ]
  },
  "order": [{ "column": "price", "direction": "asc" }],
  "limit": 20
}
```

### Disambiguation (multiple FKs)

```json
{
  "from": "orders",
  "join": {
    "billing_address": { "from": "addresses", "hint": "billing_address_id" },
    "shipping_address": { "from": "addresses", "hint": "shipping_address_id" }
  },
  "select": [
    "id",
    { "billing_address": { "select": ["street", "city"] } },
    { "shipping_address": { "select": ["street", "city"] } }
  ]
}
```

### Spread

```json
{
  "from": "films",
  "select": [
    "title",
    { "directors": {
        "spread": true,
        "select": [{ "director_name": { "column": "first_name" } }]
    }}
  ]
}
```

Result: `{ "title": "...", "director_name": "John" }` instead of `{ "title": "...", "directors": { ... } }`

### Nested embedding

```json
{
  "from": "actors",
  "select": [
    "first_name",
    { "roles": {
        "select": [
          "character",
          { "films": { "select": ["title", "year"] } }
        ]
    }}
  ]
}
```

### Join for filtering + anti-join

```json
{
  "from": "films",
  "join": {
    "actors": { "type": "inner" },
    "nominations": {}
  },
  "select": ["title"],
  "where": {
    "actors.first_name": { "$eq": "Jehanne" },
    "nominations": { "$eq": null }
  }
}
```

Films with actor "Jehanne" that have no nominations. Both joins are purely for filtering — no embed entries in `select`, so no data from those tables. `nominations: { "$eq": null }` → anti-join.

### Negation

```json
{
  "from": "products",
  "select": ["id", "name"],
  "where": {
    "status": { "$not": { "$eq": "discontinued" } },
    "name": { "$not": { "$likeAny": ["%test%", "%demo%"] } },
    "$not": {
      "$and": [
        { "price": { "$gte": 0 } },
        { "price": { "$lte": 10 } }
      ]
    }
  }
}
```

### JSON field access

```json
{
  "from": "users",
  "select": [
    "id",
    { "theme": { "column": "settings", "path": "$.theme" } },
    { "email_notif": { "column": "settings", "path": "$.notifications.email" } }
  ],
  "where": {
    "settings": {
      "path": "$.theme",
      "$eq": "dark"
    }
  }
}
```

### Safe update with strict handling

```json
{
  "type": "update",
  "from": "orders",
  "values": { "status": "shipped" },
  "where": {
    "status": { "$eq": "pending" },
    "paid": { "$is": true }
  },
  "select": ["id", "status"],
  "$meta": { "maxAffected": 100, "handling": "strict" }
}
```

### Same table, different filters

```json
{
  "from": "teams",
  "join": {
    "recent_comps": { "from": "competitions" },
    "old_comps": { "from": "competitions" }
  },
  "select": [
    "name",
    { "recent_comps": { "select": ["name"], "where": { "year": { "$gte": 2020 } } } },
    { "old_comps": { "select": ["name"], "where": { "year": { "$lt": 2020 } } } }
  ]
}
```

---

## Parser Notes

### NULL handling

- `$eq: null` → `col IS NULL`
- `$neq: null` → `col IS NOT NULL`
- `$is: null` → `col IS NULL` (explicit)
- `$is: true` → `col IS TRUE`
- `$is: false` → `col IS FALSE`
- `$is: "unknown"` → `col IS UNKNOWN`
- `$not: { $is: true }` → `col IS NOT TRUE`

### Join alias vs column name

In `where`, a key matching a join alias = embed null check (not column filter):

- `"actors": { "$eq": null }` → anti-join (no actors)
- `"actors.first_name": { "$eq": "John" }` → filter on joined column

**Rule: join aliases take precedence** over column names. Edge case if a column shares a join alias name — rename the alias.

### Dot notation in where/order

- `"actors.first_name"` in `where` → filter on joined table's column
- `"director.last_name"` in `order` → sort by joined table's column (to-one only)
- Requires matching `join` entry (explicit or implicit)

### Implicit joins

Embed entry in `select` without matching `join` entry → parser infers a relationship for the JSON aggregate subquery. No SQL JOIN is added unless the join is also used in `where`/`order`.

Explicit `join` needed only for:

- Inner join type (filter parent rows)
- FK hint (disambiguation)
- Filtering without data (join in `where` but no embed in `select`)
- Different alias than table name

### Joins without embeds

Since joins are always for filtering, a `join` entry without a matching embed in `select` is the normal case — it's a SQL JOIN used purely in `where` conditions.

Used for:

- Existence checks: `"actors": { "$neq": null }`
- Anti-joins: `"actors": { "$eq": null }`
- Joined column filters: `"actors.name": { "$eq": "..." }`

### Select entry disambiguation

Parser rule: if a select entry's value object contains a `select` field → embed entry. Otherwise → column entry.

### Group-by auto-derivation

When `group` is omitted but `select` contains aggregates alongside plain columns, auto-derive GROUP BY from all non-aggregate columns. If only aggregates in select, no GROUP BY needed.

### Computed/virtual columns

PostgREST auto-exposes functions on table types as virtual columns. Behave like regular columns in select/where/order — no special representation needed.

### JSONPath in where

```json
{ "metadata": { "path": "$.status", "$eq": "active" } }
```

→ `metadata->>'status' = 'active'` (PostgreSQL)
→ `json_extract(metadata, '$.status') = 'active'` (SQLite)

### Escaped values

JSON format handles special characters natively. Escaping only relevant when generating PostgREST URL query strings.

### Security

- Parameterize all values
- Whitelist column/table names
- Validate join aliases against schema
- Enforce `maxAffected` limits
- Validate JSONPath syntax
- Sanitize `$filter` operators

---

## Coverage

All 66+ methods mapped:

- **Core (7+):** from, select, insert, update, delete, upsert, put, join
- **Comparison (10+):** eq, neq, gt, gte, lt, lte, in, notIn, is, isDistinct + quantified (eqAny, eqAll, gtAny, gtAll, gteAny, gteAll, ltAny, ltAll, lteAny, lteAll)
- **Pattern (12):** like, ilike, likeAny, likeAll, ilikeAny, ilikeAll, regex, iregex, regexAny, regexAll, iregexAny, iregexAll
- **Array/JSON (3):** contains, containedBy, overlaps
- **Range (5):** rangeGt, rangeGte, rangeLt, rangeLte, rangeAdjacent
- **Advanced (5):** textSearch, match, or, not, filter
- **Transforms (5+):** order, limit, offset, cardinality, group
- **Response (3):** csv/geojson/explain (csv/geojson via transport)
- **Utilities (3):** abortSignal (runtime), setHeader→headers, throwOnError (runtime)
- **Specialized:** rollback, maxAffected, schema, rpc, aggregates, spread, cast, etc.

---

## Open Questions

1. ~~JSONPath in where: separate field or inline?~~ → Separate `path` field. Resolved.
2. `$match` redundant? Keeping as shorthand for multiple `$eq`.
3. `$filter` needed? Keeping as escape hatch. Security risk accepted.
4. ~~Nested `$or`/`$and`?~~ → Supported. Arbitrary nesting allowed. Resolved.
5. Ordering by aggregate aliases — PostgREST may not support directly. Need to verify.
