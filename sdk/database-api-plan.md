# Database API Documentation Plan (postgrest-js)

**Status:** ✅ Complete
**Category:** Database (1 of 5)
**Progress:** 66/66 methods documented (100%)
**Priority:** HIGH - Core functionality (DONE)

---

## Scope

Document all postgrest-js Database APIs (~66 methods) for building SQLite-based Supabase-compatible backend. Comprehensive overview, prioritized by usage frequency, theoretical SQLite compatibility analysis.

**Other Categories (TODO Later):**
- Auth (~40 methods) - signUp, signIn, MFA, OAuth, admin
- Storage (~25 methods) - buckets, files, vectors, analytics
- Realtime (~50 methods) - channels, presence, subscriptions
- Functions (~2 methods) - invoke edge functions

---

## Output Structure

Location: `/Users/dennis/Projects/docs/sdk/data/database/`

```
progress.md                        # THIS FILE - Track completion status
api-overview.md                    # Summary, categorization, priority matrix [0/1]

# HIGH Priority Files [0/15 methods]
core-operations.md                 # CRUD: select, insert, update, delete, upsert, rpc [0/6]
filters-comparison.md              # eq, neq, gt, gte, lt, lte, is, isDistinct, in, notIn [0/10]
transforms.md                      # order, limit, range, single, maybeSingle [0/5]

# MEDIUM Priority Files [0/20 methods]
filters-pattern-matching.md        # like, ilike, regexMatch variants [0/6]
filters-array-json.md              # contains, containedBy, overlaps [0/3]
filters-advanced.md                # textSearch, match, or, not, filter [0/5]
response-shaping.md                # csv, geojson, explain [0/3]
utilities.md                       # abortSignal, setHeader, throwOnError, etc [0/3]

# LOW Priority Files [0/31 methods]
filters-range.md                   # rangeGt/Gte/Lt/Lte/Adjacent (brief) [0/5]
filters-specialized.md             # Pattern variants, regex, etc (brief) [0/26]

# Reference
sqlite-compatibility-matrix.md     # Master compat table (theoretical) [0/1]
```

---

## Priority Breakdown

### HIGH Priority (15 methods) - Core 80% usage
**CRUD Operations (6):**
- `from()` - Start query
- `select()` - Fetch data
- `insert()` - Create rows
- `update()` - Modify rows
- `delete()` - Remove rows
- `upsert()` - Insert or update

**Comparison Filters (7):**
- `eq()` - Equals
- `neq()` - Not equals
- `gt()` - Greater than
- `gte()` - Greater than or equal
- `lt()` - Less than
- `lte()` - Less than or equal
- `in()` - In array

**Essential Filters (2):**
- `is()` - NULL/boolean checks
- `isDistinct()` - Distinct from

**Transforms (5):**
- `order()` - Sort results
- `limit()` - Limit rows
- `range()` - Pagination
- `single()` - Expect one row
- `maybeSingle()` - Zero or one row

### MEDIUM Priority (20 methods) - Regular 15% usage
**Pattern Filters (6):**
- `like()` - Pattern match
- `ilike()` - Case-insensitive like
- `likeAllOf()` - Match all patterns
- `likeAnyOf()` - Match any pattern
- `ilikeAllOf()` - Case-insensitive all
- `ilikeAnyOf()` - Case-insensitive any

**Array/JSON Filters (3):**
- `contains()` - Contains elements
- `containedBy()` - Contained by
- `overlaps()` - Overlaps with

**Advanced Filters (5):**
- `textSearch()` - Full-text search
- `match()` - Match all conditions
- `or()` - OR filter
- `not()` - Negate filter
- `filter()` - Custom filter

**Response Shaping (3):**
- `csv()` - CSV format
- `geojson()` - GeoJSON format
- `explain()` - Query plan

**Utilities (3):**
- `abortSignal()` - Cancel request
- `setHeader()` - Custom headers
- `throwOnError()` - Error handling

### LOW Priority (31 methods) - Specialized 5% usage
**Range Operators (5):**
- `rangeGt()`, `rangeGte()`, `rangeLt()`, `rangeLte()`, `rangeAdjacent()`

**Regex (2):**
- `regexMatch()`, `regexIMatch()`

**Set Filters (1):**
- `notIn()` - Not in array

**Control (4):**
- `rollback()`, `maxAffected()`, `returns()`, `overrideTypes()`

**Other (19):**
- Various specialized filters, chain methods, type utilities

---

## Documentation Format

### HIGH/MEDIUM Methods (Full Detail)

```markdown
#### `methodName(params)`
**Priority:** High/Medium | **SQLite:** ✅ Full / ⚠️ Partial / ❌ No

**Signature:**
```ts
method<Generics>(param: Type): ReturnType
```

**Purpose:** One-line what it does

**Parameters:**
- `param` (Type) - Description, constraints, defaults

**Returns:** ReturnType description

**Usage:**
```ts
// Common case
const { data } = await client.from('users').select().eq('id', 1)

// With chaining
const { data } = await client.from('users').select().eq('status', 'active').order('created_at')
```

**PostgREST Mapping:**
- URL param: `?column=eq.value`
- HTTP method: GET/POST/PATCH/DELETE
- Headers: Content-Type, Prefer

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full / ⚠️ Partial / ❌ Not Possible
- **SQL:** `WHERE column = ?` (example)
- **Notes:** Direct mapping / needs workaround / limitation details
- **Workarounds:** If partial/no, what's needed

**Edge Cases:**
- NULL handling: use .is() instead
- Type mismatches: throws error
- Empty strings: accepted

**Related:** Links to similar methods
```

### LOW Priority Methods (Brief)

```markdown
#### `methodName(params)`
**Priority:** Low | **SQLite:** ✅/⚠️/❌

**Signature:** `method(param: Type): ReturnType`
**Purpose:** One-line
**Usage:** `client.from('t').methodName(val)`
**SQLite:** Brief compat note
```

---

## Workflow

### Phase 1: Setup Structure
1. Create `/Users/dennis/Projects/docs/sdk/data/database/` directory
2. Create progress.md with checklist
3. Create skeleton files for all docs

### Phase 2: Extract Source Info
For each method:
1. Read source file (PostgrestFilterBuilder.ts, etc)
2. Extract method signature, JSDoc, types
3. Find corresponding test cases
4. Identify PostgREST URL param format

### Phase 3: Document HIGH Priority (15 methods)
**Files:** core-operations.md, filters-comparison.md, transforms.md

Per method:
1. Copy JSDoc → Purpose
2. Extract TypeScript sig → Signature
3. Find test examples → Usage
4. Trace URL param logic → PostgREST Mapping
5. Analyze SQL equivalent → SQLite Compatibility (theoretical)
6. Document edge cases from tests/docs
7. Add cross-references

### Phase 4: Document MEDIUM Priority (20 methods)
**Files:** filters-pattern-matching.md, filters-array-json.md, filters-advanced.md, response-shaping.md, utilities.md

Same as Phase 3, slightly less detail on edge cases

### Phase 5: Document LOW Priority (31 methods)
**Files:** filters-range.md, filters-specialized.md

Brief format only

### Phase 6: Create References
1. api-overview.md - Summary of all methods, quick reference
2. sqlite-compatibility-matrix.md - Master table of all methods w/ compat status

---

## SQLite Compatibility Analysis (Theoretical)

### ✅ Full Support (Direct SQL)
Methods with straightforward SQLite equivalents:
- Comparison filters: eq, neq, gt, gte, lt, lte
- Set filters: in, notIn
- Pattern filters: like, ilike (via COLLATE NOCASE)
- Logical: or, not, match
- Basic CRUD: select, insert, update, delete, upsert
- Transforms: order, limit, range

### ⚠️ Partial Support (Needs Workarounds)
Methods requiring additional setup/workarounds:
- `textSearch()` → SQLite FTS5 extension (different syntax)
- `contains()`, `containedBy()`, `overlaps()` → JSON functions (different API)
- `regexMatch()` → REGEXP (requires loading extension)
- `geojson()` → Custom formatting (SQLite has no native GeoJSON)
- `explain()` → EXPLAIN QUERY PLAN (different output format)

### ❌ No/Limited Support
Methods difficult/impossible in SQLite:
- Range operators (rangeGt, etc) → PostgreSQL range types, no SQLite equivalent
- `schema()` → SQLite doesn't have schema switching
- `rollback()` → PostgREST-specific transaction control
- Array operators for native arrays → SQLite has no native array type (use JSON)

---

## Critical Source Files

**Implementation reference:**
- `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/src/PostgrestClient.ts` - Main client, from/schema/rpc
- `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/src/PostgrestQueryBuilder.ts` - CRUD operations, body construction
- `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/src/PostgrestFilterBuilder.ts` - All filters, URL param logic (31 methods)
- `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/src/PostgrestTransformBuilder.ts` - Chaining, response formatting (14 methods)
- `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/src/PostgrestBuilder.ts` - HTTP execution, error handling (9 methods)
- `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/src/types/types.ts` - Type definitions

**Test references:**
- `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/test/basic.test.ts` - CRUD examples
- `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/test/filters.test.ts` - Filter examples
- `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/test/transforms.test.ts` - Transform examples

---

## Progress Tracking

Update this section after completing each file:

**Completed:** ✅ All goals achieved

- [x] api-overview.md
- [x] core-operations.md (6 methods)
- [x] filters-comparison.md (10 methods)
- [x] transforms.md (5 methods)
- [x] filters-pattern-matching.md (6 methods)
- [x] filters-array-json.md (3 methods)
- [x] filters-advanced.md (5 methods)
- [x] response-shaping.md (3 methods)
- [x] utilities.md (3 methods)
- [x] filters-range.md (5 methods, brief)
- [x] filters-specialized.md (26+ methods, brief)
- [x] sqlite-compatibility-matrix.md
- [x] progress.md

**Completion Criteria:** ✅ All Met
- [x] All 66 methods documented (varying detail by priority)
- [x] SQLite compatibility analyzed theoretically for all
- [x] Master compatibility matrix complete
- [x] Examples extracted from source
- [x] Cross-references added
- [x] 14 documentation files created

---

## Next Categories (After Database)

After completing database docs, repeat similar structure for:

1. **Auth** (supabase-js/packages/core/auth-js) - ~40 methods
2. **Storage** (supabase-js/packages/core/storage-js) - ~25 methods
3. **Realtime** (supabase-js/packages/core/realtime-js) - ~50 methods
4. **Functions** (supabase-js/packages/core/functions-js) - ~2 methods

Each category gets similar structure in sdk/data/{category}/ with priority tiers, compatibility analysis, progress tracking.
