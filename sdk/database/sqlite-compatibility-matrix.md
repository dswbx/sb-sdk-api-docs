# SQLite Compatibility Matrix

**Category:** Reference | **Status:** ✅ Complete

Theoretical analysis of implementing each postgrest-js method with SQLite backend.

---

## Legend

- ✅ **Full** - Direct SQL mapping, no workarounds needed
- ⚠️ **Partial** - Possible with extensions, manual logic, or syntax differences
- ❌ **No/Limited** - Difficult or impossible without major changes

---

## Complete Compatibility Table

### Core Operations (6/6 Full)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `from()` | ✅ Full | `FROM table_name` | Direct mapping |
| `select()` | ✅ Full | `SELECT columns` | Basic: full. Relationships: manual JOINs |
| `insert()` | ✅ Full | `INSERT INTO ... VALUES` | RETURNING requires 3.35+ |
| `update()` | ✅ Full | `UPDATE ... SET ... WHERE` | RETURNING requires 3.35+ |
| `delete()` | ✅ Full | `DELETE FROM ... WHERE` | RETURNING requires 3.35+ |
| `upsert()` | ✅ Full | `INSERT ... ON CONFLICT DO UPDATE` | Requires SQLite 3.24+ (2018) |

### Comparison Filters (10/10 Full)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `eq()` | ✅ Full | `WHERE col = ?` | Direct mapping |
| `neq()` | ✅ Full | `WHERE col != ?` | Direct mapping |
| `gt()` | ✅ Full | `WHERE col > ?` | Direct mapping |
| `gte()` | ✅ Full | `WHERE col >= ?` | Direct mapping |
| `lt()` | ✅ Full | `WHERE col < ?` | Direct mapping |
| `lte()` | ✅ Full | `WHERE col <= ?` | Direct mapping |
| `in()` | ✅ Full | `WHERE col IN (?, ?)` | Direct mapping |
| `notIn()` | ✅ Full | `WHERE col NOT IN (?, ?)` | Direct mapping |
| `is()` | ✅ Full | `WHERE col IS NULL` / `WHERE col = 1` | Booleans as INTEGER (0/1) |
| `isDistinct()` | ✅ Full | `WHERE col IS DISTINCT FROM ?` | Requires SQLite 3.39+ or manual NULL logic |

### Transforms (5/5 Full)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `order()` | ✅ Full | `ORDER BY col ASC/DESC` | NULLS FIRST/LAST requires 3.30+ |
| `limit()` | ✅ Full | `LIMIT n` | Direct mapping |
| `range()` | ✅ Full | `LIMIT n OFFSET m` | Calculate limit from range |
| `single()` | ✅ Full | Array length check (1 row) | Response handling, not SQL |
| `maybeSingle()` | ✅ Full | Array length check (0-1 rows) | Response handling, not SQL |

### Pattern Matching Filters (2 Full, 4 Partial)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `like()` | ✅ Full | `WHERE col LIKE ? COLLATE BINARY` | Case-sensitive with COLLATE BINARY |
| `ilike()` | ✅ Full | `WHERE col LIKE ? COLLATE NOCASE` | Case-insensitive with COLLATE NOCASE |
| `likeAllOf()` | ⚠️ Partial | `WHERE col LIKE ? AND col LIKE ?` | Expand array to multiple AND |
| `likeAnyOf()` | ⚠️ Partial | `WHERE col LIKE ? OR col LIKE ?` | Expand array to multiple OR |
| `ilikeAllOf()` | ⚠️ Partial | `WHERE col LIKE ? COLLATE NOCASE AND ...` | Expand + COLLATE NOCASE |
| `ilikeAnyOf()` | ⚠️ Partial | `WHERE col LIKE ? COLLATE NOCASE OR ...` | Expand + COLLATE NOCASE |

### Array & JSON Filters (0 Full, 3 Partial)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `contains()` | ⚠️ Partial | `json_each()` + EXISTS check | Requires json1 extension (built-in 3.38+) |
| `containedBy()` | ⚠️ Partial | `json_each()` + NOT EXISTS check | Requires json1 extension |
| `overlaps()` | ⚠️ Partial | `json_each()` + EXISTS check | Requires json1 extension |

**Details:**
- Store arrays as JSON: `'["a","b"]'`
- Use `json_each(column)` to iterate
- Check containment with EXISTS clauses
- For objects, use `json_extract(col, '$.key')`

### Advanced Filters (1 Full, 4 Partial)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `textSearch()` | ⚠️ Partial | FTS5 `MATCH` | Requires FTS5 virtual table, different syntax |
| `match()` | ✅ Full | Multiple `WHERE col = ? AND ...` | Direct mapping |
| `or()` | ⚠️ Partial | `WHERE (cond1 OR cond2)` | Requires PostgREST syntax parser |
| `not()` | ⚠️ Partial | `WHERE NOT (condition)` | Requires operator mapping |
| `filter()` | ⚠️ Partial | Depends on operator | Requires full operator parser |

**Details:**
- `textSearch()`: Create FTS5 table, map query syntax
- `or/not/filter()`: Parse PostgREST operator strings

### Response Shaping (1 Full, 2 Partial)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `csv()` | ✅ Full | Manual CSV formatting | Build CSV from query results |
| `geojson()` | ⚠️ Partial | SpatiaLite + manual formatting | Requires SpatiaLite for geometry |
| `explain()` | ⚠️ Partial | `EXPLAIN QUERY PLAN` | Different format, fewer options |

### Utilities (3/3 Full*)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `abortSignal()` | ✅ Full* | Check `signal.aborted` | Limited value for local SQLite |
| `setHeader()` | ✅ Full* | Pass to HTTP layer or context | N/A for local SQLite |
| `throwOnError()` | ✅ Full | Error handling pattern | Pure JS, works anywhere |

### Range Filters (0/5 - Not Supported)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `rangeGt()` | ❌ No | Separate start/end columns | No range types in SQLite |
| `rangeGte()` | ❌ No | Separate start/end columns | No range types in SQLite |
| `rangeLt()` | ❌ No | Separate start/end columns | No range types in SQLite |
| `rangeLte()` | ❌ No | Separate start/end columns | No range types in SQLite |
| `rangeAdjacent()` | ❌ No | Separate start/end columns | No range types in SQLite |

**Workaround:** Use two columns (`start`, `end`) and manual comparison:
- `rangeGt(r)`: `WHERE start > r.end`
- `rangeLt(r)`: `WHERE end < r.start`
- Overlap: `WHERE NOT (end < r.start OR start > r.end)`

### Specialized Methods (2 Full, 2 Partial, 4 No)

| Method | Compat | SQLite Implementation | Notes |
|--------|--------|----------------------|-------|
| `regexMatch()` | ⚠️ Partial | Load REGEXP extension | Not built-in |
| `regexIMatch()` | ⚠️ Partial | Load REGEXP extension + flag | Not built-in |
| `rollback()` | ❌ No | Explicit transactions | PostgREST-specific feature |
| `maxAffected()` | ✅ Full | Check `changes()` after query | Wrap in transaction |
| `returns()` | N/A | TypeScript only | Type casting |
| `overrideTypes()` | N/A | TypeScript only | Type casting |
| `schema()` | ❌ No | ATTACH DATABASE | Different concept |
| `rpc()` | ⚠️ Partial | Application functions | No stored procedures |

---

## Summary Statistics

**By Priority:**
- HIGH (21 methods): 21 ✅ Full
- MEDIUM (20 methods): 5 ✅ Full, 14 ⚠️ Partial, 1 N/A
- LOW (25+ methods): 4 ✅ Full, 4 ⚠️ Partial, 12 ❌ No, 5 N/A

**By Compatibility:**
- ✅ **Full Support: 33/66 (50%)** - Direct SQL mapping
- ⚠️ **Partial Support: 20/66 (30%)** - Possible with workarounds
- ❌ **No Support: 8/66 (12%)** - Difficult/impossible
- N/A: 5/66 (8%) - TypeScript-only

---

## Implementation Roadmap

### Phase 1: Core (High Priority) ✅
All HIGH priority methods have full SQLite support:
- Core operations: INSERT, SELECT, UPDATE, DELETE, UPSERT
- Comparison filters: =, !=, <, >, <=, >=, IN, IS
- Transforms: ORDER BY, LIMIT, OFFSET

**Effort:** Low - Direct SQL mapping

### Phase 2: Extensions (Medium Priority)
Partial support methods requiring extensions:
- **json1 extension**: Built-in since SQLite 3.38 (2022)
  - `contains()`, `containedBy()`, `overlaps()`
  - Store arrays as JSON, use `json_each()`
- **FTS5 extension**: Built-in since SQLite 3.9.0 (2015)
  - `textSearch()` - Create virtual tables, map syntax

**Effort:** Medium - Requires setup + syntax mapping

### Phase 3: Parser (Advanced Filters)
Methods requiring PostgREST syntax parser:
- `or()`, `not()`, `filter()`
- Parse operator strings: `eq.value`, `in.(a,b,c)`
- Map to SQL operators

**Effort:** Medium - Parser + operator mapping

### Phase 4: Optional Extensions
Low-priority features:
- **REGEXP extension**: Not built-in
  - `regexMatch()`, `regexIMatch()`
  - Use LIKE for simple patterns instead
- **SpatiaLite**: Geometry types
  - `geojson()` - Store as text or skip

**Effort:** High - External dependencies

### Phase 5: Skip/Workaround
Features to avoid or work around:
- **Range types**: Use separate columns
- **Schema switching**: Use ATTACH DATABASE
- **RPC**: Implement as application functions
- **rollback()**: Use explicit transactions

**Effort:** N/A - Architectural difference

---

## Key SQLite Requirements

| Feature | SQLite Version | Status |
|---------|---------------|--------|
| Basic SQL (SELECT, INSERT, etc.) | All versions | ✅ Always available |
| ON CONFLICT (upsert) | 3.24.0+ (2018) | ✅ Standard |
| NULLS FIRST/LAST | 3.30.0+ (2019) | ✅ Standard |
| RETURNING clause | 3.35.0+ (2021) | ✅ Standard |
| json1 extension | 3.38.0+ (2022) | ✅ Built-in |
| FTS5 extension | 3.9.0+ (2015) | ✅ Built-in |
| IS DISTINCT FROM | 3.39.0+ (2022) | ✅ Standard |
| REGEXP support | Extension | ⚠️ Not built-in |
| SpatiaLite | Extension | ⚠️ Separate install |

**Recommendation:** Target SQLite 3.39+ (2022) for best compatibility.

---

## See Also

- [API Overview](./api-overview.md) - All methods with links
- [Core Operations](./core-operations.md) - CRUD (full SQLite support)
- [Comparison Filters](./filters-comparison.md) - Basic filters (full support)
- [Filters - Array/JSON](./filters-array-json.md) - Requires json1 extension
