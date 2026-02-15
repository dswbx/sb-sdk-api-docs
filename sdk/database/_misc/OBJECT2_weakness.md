# OBJECT2.md Weakness Analysis vs PostgREST

## Major structural change (from Q6/Q8)

Merge `embed` into `select`. Extract `join` to top-level. Key = alias everywhere.

```json
{
  "from": "films",
  "join": {
    "actors": { "type": "inner" },
    "director": { "from": "directors", "type": "left", "hint": "director_id" }
  },
  "select": [
    "title",
    { "year_text": { "property": "year", "cast": "text" } },
    { "actors": { "select": ["name"] } },
    { "director": { "select": ["first_name", "last_name"] } }
  ],
  "where": {
    "actors.first_name": { "$eq": "Jehanne" },
    "actors": { "$neq": null }
  },
  "order": [
    { "column": "director.last_name", "direction": "desc" }
  ]
}
```

Rules:
- `join` keys are aliases. `from` only needed when alias != table name
- `join` defaults to `"left"` if omitted or if embed appears in `select` without `join` entry
- Select key = output name (alias). `property` = source column (omit if same as alias)
- Dot notation (`director.last_name`) references joined tables in `where`/`order`
- Embed-scoped `where`/`order`/`limit` stay inside the select entry

Resolves: #3, #4, #5, #11, #21.

---

## CRITICAL — affects core SQL generation

**1. Aggregate functions entirely missing**
- `avg()`, `count()`, `max()`, `min()`, `sum()` in select
- `count()` (row count) vs `column.count()` (non-null count) — different semantics
- Automatic GROUP BY when aggregates + plain columns mixed
- Casting before aggregate (`amount::numeric.sum()`) and after (`amount.avg()::int`)
- Aggregates inside embedded resources
- No representation in select format or anywhere in spec

→ Fix: add `aggregate` to select object format. Add top-level `group` field.
```json
{
  "select": [
    "category",
    { "total":     { "property": "price", "aggregate": "sum" } },
    { "avg_price": { "property": "price", "aggregate": "avg", "cast": "int" } },
    { "row_count": { "aggregate": "count" } },
    { "non_null":  { "property": "email", "aggregate": "count" } }
  ],
  "group": ["category"]
}
```
- `aggregate` without `property` → row-level (`COUNT(*)`)
- `aggregate` with `property` → column-level (`COUNT(email)`)
- `cast` before aggregate: `{ "property": "amount", "cast": "numeric", "aggregate": "sum" }`
- `cast` after aggregate: `{ "property": "price", "aggregate": "avg", "cast": "int" }`
- Q: need `castBefore`/`castAfter` distinction? Or assume cast = post-aggregate? PostgREST: before = `col::type.agg()`, after = `col.agg()::type`. Propose: `cast` = after (common), `preCast` = before (rare).
- Aggregates in embeds: same syntax inside embed's `select`

**2. Type casting missing from select format**
- PostgREST: `salary::text` — OBJECT2.md select formats have no casting syntax
- Needed for `{ "salary": { "cast": "text" } }` or similar

→ Fix: add `cast` field to select object format.
```json
{ "salary_text": { "property": "salary", "cast": "text" } }
```
Already shown in structural change above. Works standalone and with aggregates.

**3. Top-level ordering by embedded resource column not representable**
- PostgREST: `order=directors(last_name).desc` — sort parent by to-one embed's column
- Current `order` format has `column` string only, no way to reference embedded table

→ Fix: resolved by structural change. Dot notation in `order`:
```json
{ "column": "director.last_name", "direction": "desc" }
```
Requires `director` defined in `join`. Only works for to-one joins.

**4. Empty embed (filter-only) not representable**
- PostgREST: `table()` with no columns — used for existence/anti-join filtering without returning data
- No `select: []` semantics defined

→ Fix: resolved by structural change. Define in `join`, reference in `where`, omit from `select`:
```json
{
  "join": { "actors": { "type": "inner" } },
  "select": ["title"],
  "where": { "actors.name": { "$eq": "Jehanne" } }
}
```
No select entry = no data returned from that join. Clean.

--> REMARK: Kind of, joins NEVER add data to the result, that's what the embeds (now in select) are for. While parsing the request we need to identify if an embed is used for filtering, and only then

**5. Null filtering on embedded resources missing**
- `actors=not.is.null` (existence check, equivalent to `!inner`)
- `nominations=is.null` (anti-join — rows WITHOUT related records)
- `or=(actors.is.null,directors.is.null)` — OR across embeds
- None of this is representable in current `where` or `embed` structure

→ Fix: resolved by structural change. PostgREST null filtering on embeds = parent row filtering → top-level `where`:
```json
"where": {
  "actors": { "$neq": null },
  "nominations": { "$eq": null },
  "$or": [
    { "actors": { "$eq": null } },
    { "directors": { "$eq": null } }
  ]
}
```
Join name in `where` = existence/anti-join check. `$neq: null` = has related rows, `$eq: null` = no related rows.

**6. `not` prefix on operators incomplete**
- PostgREST: `not.and=(a.gte.0,a.lte.100)` — negate any compound
- OBJECT2.md `$not` only wraps a where clause; can't negate individual operators like `not.eq`, `not.like`, etc.

→ Fix: support `$not.` prefix on all operators. Both forms valid:

Prefixed (preferred — keeps them close):
```json
{ "status": { "$not.eq": "active" } }
{ "name": { "$not.like": "%test%" } }
{ "id": { "$not.in": [1, 2, 3] } }
```

Wrapper (for negating compounds):
```json
{ "$not": { "$and": [{ "a": { "$gte": 0 } }, { "a": { "$lte": 100 } }] } }
```

Full prefix list: `$not.eq`, `$not.neq`, `$not.gt`, `$not.gte`, `$not.lt`, `$not.lte`, `$not.in`, `$not.like`, `$not.ilike`, `$not.is`, `$not.regex`, `$not.iregex`, `$not.contains`, `$not.containedBy`, `$not.overlaps`, `$not.rangeGt`, etc.

## IMPORTANT — common features missing

**7. `columns` parameter for inserts**
- PostgREST: `?columns=id,bar,baz` restricts accepted payload columns
- Performance optimization for bulk inserts, prevents full-payload parsing
- Not in spec

→ Fix: add `$meta.columns`:
```json
{ "$meta": { "columns": ["id", "bar", "baz"] } }
```

**8. `missing` preference (`defaultsToNull` is partial)**
- PostgREST: `Prefer: missing=default` → use column DEFAULT instead of null
- OBJECT2.md has `defaultsToNull` boolean — inverse naming, doesn't capture `missing=default` semantics

→ Fix: rename to `$meta.missing: "null" | "default"`. Clearer, matches PostgREST naming.
- `"null"` = missing fields become NULL (current `defaultsToNull: true`)
- `"default"` = missing fields use column DEFAULT (current `defaultsToNull: false`)

**9. `handling` preference missing**
- `strict` vs `lenient` — controls whether invalid params cause errors or are silently ignored
- Affects `maxAffected` enforcement

→ Fix: add `$meta.handling: "strict" | "lenient"`.

**10. Timezone preference missing**
- `Prefer: timezone=America/Los_Angeles` — affects timestamp column output

→ Fix: add `$meta.timezone: "America/Los_Angeles"`.

**11. Multiple embeds of same table with different filters**
- PostgREST: `90_comps:competitions(name), 91_comps:competitions(name)` with different filters per alias
- Disambiguation array syntax partially handles this, but unclear for same-table different-filter case

→ Fix: resolved by structural change. Alias-keyed `join` + `select`:
```json
{
  "join": {
    "recent_comps": { "from": "competitions" },
    "old_comps":    { "from": "competitions" }
  },
  "select": [
    "name",
    { "recent_comps": { "select": ["name"], "where": { "year": { "$gte": 2020 } } } },
    { "old_comps":    { "select": ["name"], "where": { "year": { "$lt": 2020 } } } }
  ]
}
```
Each alias = separate join instance with own filters. No array syntax needed.

**12. PUT operation missing**
- PostgREST supports PUT for single-row upsert (full replace)
- `type` enum lists `upsert` but not `put` — different semantics (PUT = full replace, PATCH = partial)

→ Context: PUT = all columns replaced (missing → null/default). UPSERT (merge) = only provided columns updated, rest preserved. PUT is "overwrite row", upsert is "merge into row".

→ Fix: add `type: "put"` to operation types. Semantics: single-row, requires PK filter, full row replacement.
```json
{
  "type": "put",
  "from": "users",
  "values": { "id": 1, "name": "John", "email": "john@example.com" },
  "select": ["id", "name", "email"]
}
```
SQL: `INSERT ... ON CONFLICT(pk) DO UPDATE SET col1=excluded.col1, col2=excluded.col2, ...` (all columns, not just provided).

**13. RPC details incomplete**
- No `httpMethod` field (GET vs POST changes transaction mode based on volatility)
- No support for named vs unnamed params distinction
- No support for non-JSON input types (`text/plain`, `application/octet-stream`, `text/xml`)

→ Fix: extend RPC fields:
```json
{
  "type": "rpc",
  "function": "my_func",
  "args": { "x": 1 },
  "httpMethod": "GET",
  "paramsType": "named",
  "inputType": "json"
}
```
- `httpMethod`: `"GET"` (read-only, no transaction) | `"POST"` (default, transactional)
- `paramsType`: `"named"` (default) | `"positional"` — named = `{ "a": 1 }`, positional = `[1, 2]`
- `inputType`: `"json"` (default) | `"text"` | `"binary"` | `"xml"`

**14. `any`/`all` operator modifiers incomplete**
- PostgREST `(any)`/`(all)` works on: `eq`, `like`, `ilike`, `gt`, `gte`, `lt`, `lte`, `match`, `imatch`
- OBJECT2.md only has `$likeAnyOf`/`$likeAllOf`/`$ilikeAnyOf`/`$ilikeAnyOf`
- Missing `$eqAnyOf`, `$gtAnyOf`, `$gteAnyOf`, `$ltAnyOf`, `$lteAnyOf`, `$regexAnyOf`/`$iregexAnyOf` + `AllOf` variants

→ Fix: use camelCase `Any`/`All` suffix (each is a standalone operator):
```
$eqAny, $eqAll, $gtAny, $gtAll, $gteAny, $gteAll,
$ltAny, $ltAll, $lteAny, $lteAll,
$likeAny, $likeAll, $ilikeAny, $ilikeAll,
$regexAny, $regexAll, $iregexAny, $iregexAll
```
Replaces `$likeAnyOf` → `$likeAny`, `$ilikeAllOf` → `$ilikeAll`, etc.

Composable with `$not` wrapper: `{ "$not": { "$likeAny": [...] } }` = "doesn't match any pattern".

```json
{ "status": { "$eqAny": ["active", "pending"] } }
{ "name": { "$not": { "$likeAny": ["%test%", "%demo%"] } } }
```

## MINOR — edge cases / transport layer

**15. Stripped nulls option missing**
- `application/vnd.pgrst.array+json;nulls=stripped` — strip null keys from response
- Could be `$meta.stripNulls: true`

→ Fix: add `$meta.stripNulls: true`.

**16. Computed/virtual columns not addressed**
- PostgREST auto-exposes functions on table types as virtual columns
- Filterable, selectable, orderable — server-side concern, may not need representation

→ Fix: no spec change. Computed columns behave like regular columns in select/where/order. Parser note only.

**17. `is` operator values incomplete**
- PostgREST `is` supports: `null`, `not_null`, `true`, `false`, `unknown`
- OBJECT2.md merges `is` into `$eq`/`$neq` — works for null but `is.true`/`is.false`/`is.unknown` have no mapping

→ Fix: add `$is` operator for boolean/tristate checks:
```json
{ "active": { "$is": true } }
{ "active": { "$is": false } }
{ "status": { "$is": "unknown" } }
{ "deleted_at": { "$is": null } }
```
Keep `$eq: null` → `IS NULL` as shorthand. `$is` is explicit form covering all PostgREST `is` values.
`$not` wrapper for negation: `{ "active": { "$not": { "$is": true } } }` → `IS NOT TRUE`.

**18. Escaped/quoted filter values not addressed**
- PostgREST requires double-quote wrapping for reserved chars in values
- Parser note needed for values containing `,`, `.`, `(`, `)`, `"`

→ Fix: no spec change. JSON format handles special chars natively. Parser note: only relevant when generating PostgREST URL query strings, not for JSON→SQL path.

**19. Response format control (singular/plural)**
- `vnd.pgrst.object` → singular (covered by cardinality)
- `vnd.pgrst.array` explicit plural has no mapping

→ Fix: no change. `cardinality: "one"/"maybe"` covers singular. `"many"` (default) = array. Sufficient.

**20. `resolution` preference naming**
- PostgREST: `Prefer: resolution=merge-duplicates` vs `resolution=ignore-duplicates`
- OBJECT2.md has `ignoreDuplicates: boolean` — captures semantics but doesn't name the preference

→ Fix: keep `ignoreDuplicates`. Semantics are correct. Boolean is simpler than string enum.

## Structural issues

**21. Embed disambiguation: array syntax is irregular**
- Current: `"addresses": [{ "as": "billing_address", "hint": "..." }, ...]`
- Rest of embed is `object` keyed by table name, but disambiguation uses array — inconsistent
- Consider: always keyed by alias, with `table` field for actual table name

→ Fix: resolved by structural change. All entries alias-keyed in both `join` and `select`. No array syntax.

**22. No `from` for views vs tables distinction**
- PostgREST treats views identically to tables
- May want `"fromType": "table" | "view"` for SQL generation differences

→ Fix: no change. Views and tables use identical `from` field. SQL generation can check schema metadata if needed. Optional future: `$meta.fromType` hint, but not required.

---

## Unresolved questions (answered)

1. **Aggregate syntax** — in select format + top-level `group` field. ✅ see #1

2. **Type casting** — structured `{ "property": "salary", "cast": "text" }`. ✅ see #2

3. **Empty embed** — don't include in select, only in join+where. ✅ see #4

4. **Null filtering on embeds** — PostgREST filters PARENT rows → top-level `where`. ✅ see #5

5. **`not` prefix** — `$not.eq` preferred (close together). Wrapper `{ "$not": {...} }` for compounds. ✅ see #6

6. **Multiple same-table embeds** — merge embed into select, alias-keyed. ✅ see structural change

7. **PUT** — separate `type: "put"`, full row replacement semantics. ✅ see #12

8. **Embed ordering** — dot notation in `order`: `director.last_name`. ✅ see #3

---

## New unresolved questions (answered)

1. `cast` before vs after aggregate — `cast` = post-aggregate, `preCast` = pre-aggregate? Or single `cast` with position flag?
--> "`cast` = after (common), `preCast` = before (rare)"
  - ✅ Confirmed. PostgREST AST has separate `aggCast` (post-agg) and `cast` (on field). We use `cast` = post-agg, `preCast` = pre-agg.

2. `$not.like.any` — triple-dotted operators acceptable? Or cap at two segments?
--> this depends, how is it used in `supabase-js`? It it's used like `.not("id", "like", "any")`, then `{ "id": { "$not": {"$like": "any"} }}` is better. Otherwise `{ "id": { "$not": {"$like_any": true} }}`
  - ✅ Checked supabase-js source. `.not(col, operator, value)` → `col=not.${operator}.${value}`. Quantifier is part of operator string: `.not("col", "like(any)", "{%a%,%b%}")`. So `not` = wrapper, `like(any)` = one operator. Maps to: `{ "$not": { "$likeAny": ["%a%", "%b%"] } }`. Each quantified op is a **standalone camelCase operator** (`$likeAny`), `$not` always wraps.

3. Embed `where`/`order`/`limit` inside select entry — feels heavy. Alternative: define in `join`? But `join` is about relationship, not data shaping...
--> counterquestion: does PostgREST allow limiting/ordering embeds?
  - ✅ YES. PostgREST: `actors.order=last_name`, `actors.limit=10`, `actors.offset=2`. In AST, each child node has own `order`/`range`. **Embed-scoped where/order/limit in select entries is correct.**

4. `join` without `select` entry — should this auto-include `*` from joined table, or return nothing from it (filter-only)?
--> this depends on how PostgREST handles empty selects.
  - ✅ PostgREST `actors()` (empty select) = data omitted entirely from response. No `actors` key. Filter-only. **`join` without `select` entry = filter-only, no data.**

5. `spread` — stays in select entry? Or moves to `join`? Spread is about output shape (= select concern), not relationship.
--> `spread` stays in `select`, join only joins tables, not shapes the output.
  - ✅ Confirmed. PostgREST AST has `Spread` as a `relSelect` type (output shaping concern).

## Additional things
- align "property" and "column" (probably "column" is better? you choose)
  - **`column`**. Consistent with `order: [{ "column": "price" }]`, matches PostgREST AST `field.name`. Format: `{ "alias": { "column": "source_col", "cast": "text" } }`. Omit `column` when alias == column name: `{ "salary": { "cast": "text" } }`.
- I've added `./postgrest-plan-ast.json`, please take a look at it, maybe it helps thinking about our AST structure, maybe we need to change or there might be things we missed.
  - ✅ Full analysis below.

---

## PostgREST AST analysis (`postgrest-plan-ast.json`)

### Structure comparison

PostgREST AST is a **tree** — each node = complete query context (`select`, `where`, `order`, `range`), children = embedded resources. Our format is **flat with nesting in select**. Both work; ours is more compact.

### Select entry fields (AST)

```
field: { name, jsonPath[], toJson, toTsVector, irType, baseType, transform, default, fullRow }
aggFunction: Sum | Avg | Max | Min | Count | null
aggCast: string | null          ← cast AFTER aggregate
cast: string | null             ← cast on field (before aggregate if both present)
alias: string | null
```

- Maps to our: `{ "alias": { "column": "name", "aggregate": "sum", "cast": "int", "preCast": "numeric", "path": "$.key" } }`

### Node/relationship fields (AST)

```
relName: "authors"              ← actual table name
relAlias: "author"              ← alias (null if same)
relJoinType: JTInner | JTLeft
relHint: string | null          ← FK hint for disambiguation
relSpread: null | SpreadType
relToParent.cardinality: O2M | M2O | O2O | M2M
relJoinConds: [{ source.field, target.field }]
relSelect: [JsonEmbed | Spread] ← which embeds this node includes
emptyEmbed: boolean             ← true = filter-only
```

- Maps to our `join`: `{ "from": "authors", "type": "inner", "hint": "author_id" }`
- `emptyEmbed` = derived from whether embed appears in `select`

### Filter structure (AST)

```
CoercibleExpr: { negate: bool, operator: And|Or, conditions[] }    ← logical
CoercibleStmnt: { filter: CoercibleFilter }                        ← leaf
CoercibleFilter: { field, opExpr: { negate: bool, operation } }    ← column filter
CoercibleFilterNullEmbed                                           ← embed null check (special!)
```

Operations:
- `OpQuant`: `{ operator: OpEqual|OpLike|..., quantifier: null|QuantAny|QuantAll, value }`
- `In`: `{ value[] }`
- `Is`: `IsNull | IsNotNull | IsTriTrue | IsTriFalse | IsTriUnknown`
- `Fts`: `{ operator: FilterFts|..., language, value }`
- `Op`: simple operators (contains, overlap, range ops)

- `negate` on opExpr = our `$not` wrapper
- `quantifier` on OpQuant = our `Any`/`All` camelCase suffix (`$likeAny`, `$eqAll`, etc.)
- `CoercibleFilterNullEmbed` = our `"actors": { "$eq": null }` in where. **Parser rule: if key matches a join alias → embed null check, not column filter.**

### RPC structure (AST)

```
funCParams: KeyParams | OnePosParam     ← named vs positional
funCArgs: DirectArgs | JsonArgs
routine.volatility: Volatile | Stable | Immutable
routine.returnType: Single | SetOf
```

- `KeyParams` = named, `OnePosParam` = positional. Maps to our `paramsType`.
- `Stable`/`Immutable` → safe for GET. `Volatile` → POST. Our `httpMethod` or let server decide from volatility.

### Mutation structure (AST)

```
Insert: { insCols[], insBody, onConflict: { resolution, columns }, returning[], insPkCols[], applyDefs }
```

- `applyDefs` = our `$meta.missing: "default"`
- `returning` = our `select` on mutations

### Things the AST has that we don't need

- `irType`/`baseType` — type resolution is server-side
- `toJson` — transport concern
- `embedMode: JsonObject | JsonArray` — determined by cardinality, server-side
- `relJoinConds` — FK column mappings resolved by server from `hint`
- `insPkCols` — PK detection is server-side

### Things to reconsider

1. **`$not` as wrapper only** — PostgREST uses `negate: bool` on opExpr, supabase-js uses `.not(col, op, val)`. Both treat not as wrapper. Propose: **drop `$not.eq` prefix form from #6**, use only wrapper `{ "$not": { "$eq": ... } }`. Simpler parser, matches both PostgREST internals and supabase-js API. Update #6 fix accordingly.

2. **Embed null check ambiguity** — `"actors": { "$eq": null }` in where: is `actors` a column or join alias? Parser rule: **join aliases take precedence**. If you need to filter a column named `actors`, use `$meta` disambiguation or ensure table/column naming doesn't collide. Edge case, acceptable tradeoff.

3. **`SpreadType` distinction** — AST has `ToOneSpread | ToManySpread`. To-one spread flattens object fields into parent. To-many spread... flattens array? Probably not useful. Keep `spread: boolean` for now, only meaningful for to-one relationships.

4. **Revised #6 fix** — with `$not` as wrapper only:
```json
{ "status": { "$not": { "$eq": "active" } } }
{ "name": { "$not": { "$likeAny": ["%test%", "%demo%"] } } }
{ "$not": { "$and": [{ "a": { "$gte": 0 } }, { "a": { "$lte": 100 } }] } }
```
Single negation mechanism. Each quantified op is a standalone operator (`$likeAny`, `$eqAll`). `$not` always wraps.
