# Database API Methods Overview

Quick reference: all 66 methods with PostgREST params, signatures, descriptions, and links.

---

## HIGH Priority (21 methods)

### Core Operations (6) [core-operations.md](./core-operations.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `from(relation)` | `/relation` | Start query on table/view |
| `select(columns?, options?)` | `?select=cols` | Fetch rows from table/view |
| `insert(values, options?)` | POST with body | Insert new rows into table |
| `update(values, options?)` | PATCH with body | Modify existing rows in table |
| `delete(options?)` | DELETE request | Remove rows from table |
| `upsert(values, options?)` | POST with body | Insert or update on conflict |

### Comparison Filters (10) [filters-comparison.md](./filters-comparison.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `eq(column, value)` | `?col=eq.value` | Match rows where column equals value |
| `neq(column, value)` | `?col=neq.value` | Match rows where column is not equal to value |
| `gt(column, value)` | `?col=gt.value` | Match rows where column is greater than value |
| `gte(column, value)` | `?col=gte.value` | Match rows where column is greater than or equal to value |
| `lt(column, value)` | `?col=lt.value` | Match rows where column is less than value |
| `lte(column, value)` | `?col=lte.value` | Match rows where column is less than or equal to value |
| `in(column, values)` | `?col=in.(val1,val2)` | Match rows where column value is in array |
| `notIn(column, values)` | `?col=not.in.(val1,val2)` | Match rows where column value is NOT in array |
| `is(column, value)` | `?col=is.null` | Match rows where column IS value (NULL checks) |
| `isDistinct(column, value)` | `?col=isdistinct.value` | Match rows where column IS DISTINCT FROM value (NULL-aware) |

### Transforms (5) [transforms.md](./transforms.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `order(column, options?)` | `?order=col.asc` | Sort query results by column |
| `limit(count, options?)` | `?limit=10` | Limit number of rows returned |
| `range(from, to, options?)` | `?offset=10&limit=10` | Paginate results using inclusive range |
| `single()` | Accept header | Return data as single object (requires exactly 1 row) |
| `maybeSingle()` | Accept header | Return data as single object or null (allows 0 or 1 row) |

---

## MEDIUM Priority (20 methods)

### Pattern Matching Filters (6) [filters-pattern-matching.md](./filters-pattern-matching.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `like(column, pattern)` | `?col=like.pattern` | Match rows where column matches SQL LIKE pattern (case-sensitive) |
| `ilike(column, pattern)` | `?col=ilike.pattern` | Match rows where column matches SQL LIKE pattern (case-insensitive) |
| `likeAllOf(column, patterns)` | `?col=like(all).{...}` | Match rows where column matches ALL patterns (case-sensitive) |
| `likeAnyOf(column, patterns)` | `?col=like(any).{...}` | Match rows where column matches ANY pattern (case-sensitive) |
| `ilikeAllOf(column, patterns)` | `?col=ilike(all).{...}` | Match rows where column matches ALL patterns (case-insensitive) |
| `ilikeAnyOf(column, patterns)` | `?col=ilike(any).{...}` | Match rows where column matches ANY pattern (case-insensitive) |

### Array & JSON Filters (3) [filters-array-json.md](./filters-array-json.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `contains(column, value)` | `?col=cs.{...}` | Match rows where column contains all elements in value |
| `containedBy(column, value)` | `?col=cd.{...}` | Match rows where every element in column is contained by value |
| `overlaps(column, value)` | `?col=ov.{...}` | Match rows where column and value have at least one element in common |

### Advanced Filters (5) [filters-advanced.md](./filters-advanced.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `textSearch(column, query, options?)` | `?col=fts.query` | Full-text search on text or tsvector columns |
| `match(query)` | `?col1=eq.val1&col2=eq.val2` | Match multiple columns with equality (shorthand for multiple eq()) |
| `or(filters, options?)` | `?or=(cond1,cond2)` | Match rows satisfying at least one filter condition (logical OR) |
| `not(column, operator, value)` | `?col=not.op.value` | Negate filter condition (match rows NOT satisfying filter) |
| `filter(column, operator, value)` | `?col=op.value` | Escape hatch for custom filters using raw PostgREST operators |

### Response Shaping (3) [response-shaping.md](./response-shaping.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `csv()` | Accept: text/csv | Return query results as CSV text instead of JSON |
| `geojson()` | Accept: geo+json | Return query results in GeoJSON format for geographic data |
| `explain(options?)` | Accept: plan header | Return query execution plan instead of results (performance debugging) |

### Utilities (3) [utilities.md](./utilities.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `abortSignal(signal)` | Passed to fetch() | Set AbortSignal for request cancellation |
| `setHeader(name, value)` | Custom header | Set custom HTTP header for request |
| `throwOnError()` | Error handling | Change error handling from returned error to thrown exception |

---

## LOW Priority (25 methods)

### Range Filters (5) [filters-range.md](./filters-range.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `rangeGt(column, range)` | `?col=sr.range` | Match rows where every element in column range is greater than any element in range |
| `rangeGte(column, range)` | `?col=nxl.range` | Match rows where column range is either contained in or greater than range |
| `rangeLt(column, range)` | `?col=sl.range` | Match rows where every element in column range is less than any element in range |
| `rangeLte(column, range)` | `?col=nxr.range` | Match rows where column range is either contained in or less than range |
| `rangeAdjacent(column, range)` | `?col=adj.range` | Match rows where column range is adjacent to range (mutually exclusive, no gap) |

### Specialized Methods (20+) [filters-specialized.md](./filters-specialized.md)

| Method | PostgREST Param | Description |
|--------|-----------------|-------------|
| `regexMatch(column, pattern)` | `?col=~.pattern` | Case-sensitive regex match using PostgreSQL ~ operator |
| `regexIMatch(column, pattern)` | `?col=~*.pattern` | Case-insensitive regex match using PostgreSQL ~* operator |
| `rollback()` | Prefer: tx=rollback | Rollback query transaction (data returned but not committed) |
| `maxAffected(value)` | Prefer: max-affected | Set max rows that can be affected by UPDATE/DELETE |
| `returns<NewResult>()` | N/A (TypeScript) | Override TypeScript return type (deprecated, use overrideTypes) |
| `overrideTypes<NewResult, Options>()` | N/A (TypeScript) | Override TypeScript return type with merge option |
| `schema(schemaName)` | Accept-Profile header | Switch to different PostgreSQL schema |
| `rpc(fn, args?, options?)` | POST /rpc/fn | Call PostgreSQL stored procedure/function |

---

## See Also

- [README.md](./README.md) - Database API documentation navigation guide
- [api-overview.md](./api-overview.md) - High-level API overview with basic method table
- [sqlite-compatibility-matrix.md](./sqlite-compatibility-matrix.md) - SQLite compatibility analysis and implementation roadmap
- [progress.md](./progress.md) - Documentation completion status

---

## Notes

- **PostgREST Param**: Shows URL parameter or HTTP header mapping
- **Simplified Signatures**: Complex generics removed for readability (see detailed docs for full types)
- **SQLite Compatibility**: See [sqlite-compatibility-matrix.md](./sqlite-compatibility-matrix.md) for implementation details
- **Priority Tiers**: HIGH (~80% usage), MEDIUM (~15%), LOW (~5%)
- **Link Format**: Uses lowercase with hyphens for markdown anchors
