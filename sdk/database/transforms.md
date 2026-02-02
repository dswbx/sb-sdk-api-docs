# Transforms

**Category:** HIGH Priority | **Status:** ✅ Complete
**Methods:** 5 (order, limit, range, single, maybeSingle)

Methods for transforming query results: sorting, pagination, and response shape.

---

## `order(column, options)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
order<ColumnName extends string & keyof Row>(
  column: ColumnName,
  options?: {
    ascending?: boolean
    nullsFirst?: boolean
    referencedTable?: string
  }
): this
```

**Purpose:** Sort query results by column.

**Parameters:**
- `column` (string) - Column name to sort by
- `options.ascending` (boolean, optional) - Sort direction. Default: `true`
  - `true`: Ascending (A-Z, 0-9, oldest-newest)
  - `false`: Descending (Z-A, 9-0, newest-oldest)
- `options.nullsFirst` (boolean, optional) - NULL placement. Default: depends on ascending
  - `true`: NULL values appear first
  - `false`: NULL values appear last
  - Unspecified: Database default (NULLs last for ASC, first for DESC in Postgres)
- `options.referencedTable` (string, optional) - Sort related table instead of parent

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Ascending (default)
const { data } = await client
  .from('users')
  .select()
  .order('name')

// Descending
const { data } = await client
  .from('products')
  .select()
  .order('price', { ascending: false })

// Multiple sorts (order matters!)
const { data } = await client
  .from('users')
  .select()
  .order('status', { ascending: true })
  .order('created_at', { ascending: false })

// NULL handling
const { data } = await client
  .from('users')
  .select()
  .order('last_login', { ascending: false, nullsFirst: false })

// Sort related table
const { data } = await client
  .from('users')
  .select('*, posts(*)')
  .order('created_at', { referencedTable: 'posts' })
```

**PostgREST Mapping:**
- URL param: `?order=name.asc`
- URL param: `?order=price.desc`
- URL param (multiple): `?order=status.asc,created_at.desc`
- URL param (nulls): `?order=last_login.desc.nullslast`
- URL param (related): `?posts.order=created_at.desc`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `ORDER BY name ASC`
- **SQL (desc):** `ORDER BY price DESC`
- **SQL (multiple):** `ORDER BY status ASC, created_at DESC`
- **SQL (nulls):** `ORDER BY last_login DESC NULLS LAST`
- **Notes:**
  - Direct mapping to ORDER BY clause
  - NULLS FIRST/LAST supported in SQLite 3.30+ (2019)
  - For older SQLite, use: `ORDER BY (column IS NULL), column DESC`
  - Related table sorting: Requires JOIN awareness
- **Workarounds:**
  - SQLite < 3.30: `ORDER BY (last_login IS NULL) ASC, last_login DESC` for NULLS LAST

**Edge Cases:**
- Multiple `.order()` calls: Appends to existing order (compound sort)
- Order matters: `.order('a').order('b')` sorts by a first, then b
- Non-existent column: Database error on execution
- Related table ordering: Only affects parent if using `!inner` join

**Related:** limit(), range()

**Source:** PostgrestTransformBuilder.ts:75-139

---

## `limit(count, options)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
limit(
  count: number,
  options?: {
    referencedTable?: string
  }
): this
```

**Purpose:** Limit number of rows returned.

**Parameters:**
- `count` (number) - Maximum rows to return. Must be non-negative.
- `options.referencedTable` (string, optional) - Limit related table instead of parent

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Limit results
const { data } = await client
  .from('users')
  .select()
  .limit(10)

// With filter
const { data } = await client
  .from('products')
  .select()
  .eq('category', 'electronics')
  .limit(20)

// With order (pagination pattern)
const { data } = await client
  .from('posts')
  .select()
  .order('created_at', { ascending: false })
  .limit(50)

// Limit related table
const { data } = await client
  .from('users')
  .select('*, posts(*)')
  .limit(5, { referencedTable: 'posts' })
// Returns all users, but only 5 posts per user
```

**PostgREST Mapping:**
- URL param: `?limit=10`
- URL param (related): `?posts.limit=5`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `LIMIT 10`
- **Notes:**
  - Direct mapping to LIMIT clause
  - Related table limiting: Requires subquery or window function
- **Implementation:** `SELECT * FROM users LIMIT 10`

**Edge Cases:**
- `limit(0)`: Returns no rows (valid query)
- Negative count: Type error (TypeScript) or database error
- count > total rows: Returns all available rows
- Related table limit: Requires careful query construction

**Related:** range(), order(), offset

**Source:** PostgrestTransformBuilder.ts:141-161

---

## `range(from, to, options)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
range(
  from: number,
  to: number,
  options?: {
    referencedTable?: string
  }
): this
```

**Purpose:** Paginate results using inclusive range. Zero-indexed.

**Parameters:**
- `from` (number) - Starting index (inclusive, zero-based)
- `to` (number) - Ending index (inclusive, zero-based)
- `options.referencedTable` (string, optional) - Paginate related table instead

**Returns:** Same builder for chaining.

**Usage:**
```ts
// First page (rows 0-9)
const { data } = await client
  .from('users')
  .select()
  .range(0, 9)

// Second page (rows 10-19)
const { data } = await client
  .from('users')
  .select()
  .range(10, 19)

// Single row (row 5)
const { data } = await client
  .from('users')
  .select()
  .range(5, 5)

// With order (crucial for stable pagination!)
const { data } = await client
  .from('posts')
  .select()
  .order('created_at', { ascending: false })
  .range(0, 49) // 50 items per page

// Calculate range from page number
const page = 2
const pageSize = 20
const from = page * pageSize
const to = from + pageSize - 1
const { data } = await client
  .from('users')
  .select()
  .range(from, to) // Returns rows 40-59
```

**PostgREST Mapping:**
- URL param: `?offset=10&limit=10` (for range(10, 19))
- Range is inclusive, so `range(0, 9)` becomes `offset=0&limit=10`
- Calculation: `offset = from`, `limit = to - from + 1`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `LIMIT 10 OFFSET 10`
- **Notes:**
  - Direct mapping to LIMIT + OFFSET
  - Range is inclusive, so adjust limit: `LIMIT (to - from + 1) OFFSET from`
- **Implementation:** `SELECT * FROM users LIMIT 10 OFFSET 0` (for range(0, 9))

**Edge Cases:**
- `from > to`: Returns empty result
- `from = to`: Returns single row (if exists)
- Negative values: Type error or database error
- Without order: Results may be inconsistent across pages (database-dependent ordering)
- Beyond total rows: Returns partial results or empty array

**Related:** limit(), order()

**Source:** PostgrestTransformBuilder.ts:163-193

---

## `single()`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
single<ResultOne = Result extends (infer ResultOne)[] ? ResultOne : never>():
  PostgrestBuilder<ResultOne>
```

**Purpose:** Return data as single object instead of array. Requires exactly 1 row.

**Parameters:** None

**Returns:** Builder with result type changed from `T[]` to `T`.

**Usage:**
```ts
// Expect single row by ID
const { data, error } = await client
  .from('users')
  .select()
  .eq('id', 1)
  .single()
// data: User | null (not User[] | null)

// With limit (recommended)
const { data, error } = await client
  .from('users')
  .select()
  .eq('email', 'john@example.com')
  .limit(1)
  .single()

// Error cases
const { data, error } = await client
  .from('users')
  .select()
  .eq('status', 'active')
  .single()
// Error if 0 rows: "JSON object requested, multiple (or no) rows returned"
// Error if 2+ rows: "JSON object requested, multiple (or no) rows returned"
```

**PostgREST Mapping:**
- Header: `Accept: application/vnd.pgrst.object+json`
- Forces single-object response
- Error codes:
  - 406 (Not Acceptable): 0 rows or 2+ rows returned

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** Same query, handle response differently
- **Notes:**
  - No SQL-level change, only response handling
  - Check result array length:
    - 0 rows: Return error
    - 1 row: Return row as object (not array)
    - 2+ rows: Return error
- **Implementation:**
  ```ts
  const rows = executeQuery('SELECT * FROM users WHERE id = ?')
  if (rows.length === 0) throw Error('No rows')
  if (rows.length > 1) throw Error('Multiple rows')
  return rows[0]
  ```

**Edge Cases:**
- 0 rows: Returns error (use `.maybeSingle()` to allow 0)
- 2+ rows: Returns error (add `.limit(1)` to prevent)
- Always use with filters that guarantee single result
- Recommended: Combine with `.limit(1)` for safety

**Related:** maybeSingle(), limit()

**Source:** PostgrestTransformBuilder.ts:205-217

---

## `maybeSingle()`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
maybeSingle<ResultOne = Result extends (infer ResultOne)[] ? ResultOne : never>():
  PostgrestBuilder<ResultOne | null>
```

**Purpose:** Return data as single object or null. Allows 0 or 1 row.

**Parameters:** None

**Returns:** Builder with result type changed from `T[]` to `T | null`.

**Usage:**
```ts
// Optional single row
const { data, error } = await client
  .from('users')
  .select()
  .eq('email', 'john@example.com')
  .maybeSingle()
// data: User | null
// No error if 0 rows (data = null)

// With limit (recommended)
const { data, error } = await client
  .from('users')
  .select()
  .eq('username', 'john')
  .limit(1)
  .maybeSingle()

// Error case
const { data, error } = await client
  .from('users')
  .select()
  .eq('status', 'active')
  .maybeSingle()
// Error if 2+ rows: "JSON object requested, multiple (or no) rows returned"
```

**PostgREST Mapping:**
- Header: `Accept: application/vnd.pgrst.object+json` (non-GET)
- Header: `Accept: application/json` (GET, with special handling)
- Allows 0 or 1 row
- Error codes:
  - 406 (Not Acceptable): 2+ rows returned

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** Same query, handle response differently
- **Notes:**
  - No SQL-level change, only response handling
  - Check result array length:
    - 0 rows: Return null
    - 1 row: Return row as object
    - 2+ rows: Return error
- **Implementation:**
  ```ts
  const rows = executeQuery('SELECT * FROM users WHERE email = ?')
  if (rows.length === 0) return null
  if (rows.length > 1) throw Error('Multiple rows')
  return rows[0]
  ```

**Edge Cases:**
- 0 rows: Returns `null` (no error)
- 1 row: Returns object
- 2+ rows: Returns error (add `.limit(1)` to prevent)
- Useful for "find by unique key" operations where row may not exist
- Recommended: Combine with `.limit(1)` for safety

**Related:** single(), limit()

**Source:** PostgrestTransformBuilder.ts:219-237

---

## Summary

All 5 transform methods have full SQLite compatibility:

| Method | Purpose | SQLite Mapping | Notes |
|--------|---------|----------------|-------|
| `order()` | Sort results | `ORDER BY column ASC/DESC` | Direct mapping; NULLS FIRST/LAST in 3.30+ |
| `limit()` | Limit rows | `LIMIT count` | Direct mapping |
| `range()` | Paginate | `LIMIT (to-from+1) OFFSET from` | Inclusive range |
| `single()` | Get one object | Array length check | Response handling, not SQL |
| `maybeSingle()` | Get one or null | Array length check | Response handling, not SQL |

**Best Practices:**
- Always use `.order()` before `.range()` for stable pagination
- Use `.limit(1)` with `.single()` or `.maybeSingle()` to prevent errors
- Use `.single()` when row must exist, `.maybeSingle()` when optional
