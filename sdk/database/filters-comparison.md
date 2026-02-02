# Comparison Filters

**Category:** HIGH Priority | **Status:** ✅ Complete
**Methods:** 10 (eq, neq, gt, gte, lt, lte, in, notIn, is, isDistinct)

Basic comparison operators for filtering query results.

---

## `eq(column, value)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
eq<ColumnName extends string>(
  column: ColumnName,
  value: NonNullable<Row[ColumnName]>
): this
```

**Purpose:** Match rows where column equals value.

**Parameters:**
- `column` (string) - Column name to filter on
- `value` (any) - Value to match (type-safe based on schema). Cannot be NULL.

**Returns:** Same builder for chaining.

**Usage:**
```ts
// String equality
const { data } = await client
  .from('users')
  .select()
  .eq('name', 'John')

// Number equality
const { data } = await client
  .from('users')
  .select()
  .eq('id', 42)

// Boolean equality
const { data } = await client
  .from('users')
  .select()
  .eq('is_active', true)

// Multiple filters (AND)
const { data } = await client
  .from('users')
  .select()
  .eq('status', 'active')
  .eq('role', 'admin')

// For NULL, use .is() instead
const { data } = await client
  .from('users')
  .select()
  .is('deleted_at', null)
```

**PostgREST Mapping:**
- URL param: `?name=eq.John`
- URL param: `?id=eq.42`
- Multiple filters: `?status=eq.active&role=eq.admin`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE name = ?`
- **Notes:** Direct mapping to `=` operator
- **Implementation:** `SELECT * FROM users WHERE name = 'John'`

**Edge Cases:**
- NULL value: Use `.is(column, null)` instead
- Empty string: Valid, matches empty string
- Case sensitivity: Case-sensitive for strings (use `.ilike()` for case-insensitive)
- Type mismatch: Database handles coercion or errors

**Related:** neq(), is(), in()

**Source:** PostgrestFilterBuilder.ts:96-117

---

## `neq(column, value)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
neq<ColumnName extends string>(
  column: ColumnName,
  value: Row[ColumnName]
): this
```

**Purpose:** Match rows where column is not equal to value.

**Parameters:**
- `column` (string) - Column name to filter on
- `value` (any) - Value to exclude

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Not equal
const { data } = await client
  .from('users')
  .select()
  .neq('status', 'deleted')

// Multiple exclusions (AND)
const { data } = await client
  .from('users')
  .select()
  .neq('role', 'guest')
  .neq('status', 'banned')
```

**PostgREST Mapping:**
- URL param: `?status=neq.deleted`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE status != ? OR status IS NULL`
- **Notes:** Direct mapping to `!=` or `<>` operator
- **Implementation:** `SELECT * FROM users WHERE status != 'deleted'`

**Edge Cases:**
- NULL behavior: NULL != 'value' returns NULL (neither true nor false). Use `.not('column', 'is', null)` to exclude NULLs explicitly.
- Empty string: Valid exclusion

**Related:** eq(), notIn(), isDistinct()

**Source:** PostgrestFilterBuilder.ts:119-135

---

## `gt(column, value)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
gt<ColumnName extends string & keyof Row>(
  column: ColumnName,
  value: Row[ColumnName]
): this
```

**Purpose:** Match rows where column is greater than value.

**Parameters:**
- `column` (string) - Column name to filter on
- `value` (any) - Value to compare against

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Numbers
const { data } = await client
  .from('products')
  .select()
  .gt('price', 100)

// Dates
const { data } = await client
  .from('events')
  .select()
  .gt('created_at', '2024-01-01')

// With other filters
const { data } = await client
  .from('products')
  .select()
  .gt('price', 50)
  .lt('price', 200)
```

**PostgREST Mapping:**
- URL param: `?price=gt.100`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE price > ?`
- **Notes:** Direct mapping to `>` operator
- **Implementation:** `SELECT * FROM products WHERE price > 100`

**Edge Cases:**
- String comparison: Lexicographical order
- NULL values: NULL > value returns NULL (excluded from results)
- Date strings: Compared as strings (works for ISO 8601 format)

**Related:** gte(), lt(), lte()

**Source:** PostgrestFilterBuilder.ts:137-148

---

## `gte(column, value)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
gte<ColumnName extends string & keyof Row>(
  column: ColumnName,
  value: Row[ColumnName]
): this
```

**Purpose:** Match rows where column is greater than or equal to value.

**Parameters:**
- `column` (string) - Column name to filter on
- `value` (any) - Value to compare against (inclusive)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Numbers
const { data } = await client
  .from('products')
  .select()
  .gte('price', 100)

// Dates (inclusive range)
const { data } = await client
  .from('events')
  .select()
  .gte('start_date', '2024-01-01')
  .lt('start_date', '2024-02-01')
```

**PostgREST Mapping:**
- URL param: `?price=gte.100`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE price >= ?`
- **Notes:** Direct mapping to `>=` operator
- **Implementation:** `SELECT * FROM products WHERE price >= 100`

**Edge Cases:**
- Same as `gt()` but includes equal values

**Related:** gt(), lte(), lt()

**Source:** PostgrestFilterBuilder.ts:150-161

---

## `lt(column, value)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
lt<ColumnName extends string & keyof Row>(
  column: ColumnName,
  value: Row[ColumnName]
): this
```

**Purpose:** Match rows where column is less than value.

**Parameters:**
- `column` (string) - Column name to filter on
- `value` (any) - Value to compare against

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Numbers
const { data } = await client
  .from('products')
  .select()
  .lt('stock', 10)

// Dates
const { data } = await client
  .from('users')
  .select()
  .lt('created_at', '2020-01-01')
```

**PostgREST Mapping:**
- URL param: `?stock=lt.10`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE stock < ?`
- **Notes:** Direct mapping to `<` operator
- **Implementation:** `SELECT * FROM products WHERE stock < 10`

**Edge Cases:**
- Same as `gt()` but opposite direction

**Related:** lte(), gt(), gte()

**Source:** PostgrestFilterBuilder.ts:163-174

---

## `lte(column, value)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
lte<ColumnName extends string & keyof Row>(
  column: ColumnName,
  value: Row[ColumnName]
): this
```

**Purpose:** Match rows where column is less than or equal to value.

**Parameters:**
- `column` (string) - Column name to filter on
- `value` (any) - Value to compare against (inclusive)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Numbers
const { data } = await client
  .from('products')
  .select()
  .lte('price', 500)

// Dates (inclusive range)
const { data } = await client
  .from('events')
  .select()
  .gte('date', '2024-01-01')
  .lte('date', '2024-12-31')
```

**PostgREST Mapping:**
- URL param: `?price=lte.500`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE price <= ?`
- **Notes:** Direct mapping to `<=` operator
- **Implementation:** `SELECT * FROM products WHERE price <= 500`

**Edge Cases:**
- Same as `lt()` but includes equal values

**Related:** lt(), gte(), gt()

**Source:** PostgrestFilterBuilder.ts:176-187

---

## `in(column, values)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
in<ColumnName extends string>(
  column: ColumnName,
  values: ReadonlyArray<Row[ColumnName]>
): this
```

**Purpose:** Match rows where column value is in the provided array.

**Parameters:**
- `column` (string) - Column name to filter on
- `values` (array) - Array of values to match against

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Array of strings
const { data } = await client
  .from('users')
  .select()
  .in('status', ['active', 'pending'])

// Array of numbers
const { data } = await client
  .from('products')
  .select()
  .in('id', [1, 2, 3, 4, 5])

// Array with special characters (auto-quoted)
const { data } = await client
  .from('users')
  .select()
  .in('name', ['John (Admin)', 'Jane,Doe'])

// Empty array
const { data } = await client
  .from('users')
  .select()
  .in('id', []) // Returns no results
```

**PostgREST Mapping:**
- URL param: `?status=in.(active,pending)`
- URL param (numbers): `?id=in.(1,2,3,4,5)`
- Reserved chars: Auto-quoted: `?name=in.("John (Admin)","Jane,Doe")`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE status IN (?, ?)`
- **Notes:** Direct mapping to `IN` operator
- **Implementation:** `SELECT * FROM users WHERE status IN ('active', 'pending')`

**Edge Cases:**
- Empty array: Returns no results (equivalent to `WHERE false`)
- Single value: Same as `.eq()`
- Duplicate values: Auto-deduplicated
- NULL in array: NULL handling follows SQL IN semantics
- Reserved characters (`,`, `(`, `)`): Auto-quoted with `"`

**Related:** notIn(), eq(), or()

**Source:** PostgrestFilterBuilder.ts:351-380

---

## `notIn(column, values)`

**Priority:** High | **SQLite:** ✅ Full (moved from LOW to HIGH based on usage)

**Signature:**
```ts
notIn<ColumnName extends string>(
  column: ColumnName,
  values: ReadonlyArray<Row[ColumnName]>
): this
```

**Purpose:** Match rows where column value is NOT in the provided array.

**Parameters:**
- `column` (string) - Column name to filter on
- `values` (array) - Array of values to exclude

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Exclude multiple values
const { data } = await client
  .from('users')
  .select()
  .notIn('status', ['deleted', 'banned'])

// Exclude IDs
const { data } = await client
  .from('products')
  .select()
  .notIn('id', [1, 2, 3])
```

**PostgREST Mapping:**
- URL param: `?status=not.in.(deleted,banned)`
- Reserved chars handling: Same as `in()`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE status NOT IN (?, ?)`
- **Notes:** Direct mapping to `NOT IN` operator
- **Implementation:** `SELECT * FROM users WHERE status NOT IN ('deleted', 'banned')`

**Edge Cases:**
- Empty array: Returns all results (equivalent to no filter)
- NULL handling: NULL NOT IN (values) returns NULL (excluded)
- Duplicate values: Auto-deduplicated

**Related:** in(), neq()

**Source:** PostgrestFilterBuilder.ts:382-408

---

## `is(column, value)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
is<ColumnName extends string & keyof Row>(
  column: ColumnName,
  value: boolean | null
): this
```

**Purpose:** Match rows where column IS value. Primarily for NULL checks and boolean columns.

**Parameters:**
- `column` (string) - Column name to filter on
- `value` (boolean | null) - Value to match:
  - `null`: Check if column IS NULL
  - `true`: Check if boolean column is TRUE
  - `false`: Check if boolean column is FALSE

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Check NULL
const { data } = await client
  .from('users')
  .select()
  .is('deleted_at', null)

// Check NOT NULL (use .not())
const { data } = await client
  .from('users')
  .select()
  .not('deleted_at', 'is', null)

// Boolean true
const { data } = await client
  .from('users')
  .select()
  .is('is_active', true)

// Boolean false
const { data } = await client
  .from('users')
  .select()
  .is('is_verified', false)
```

**PostgREST Mapping:**
- URL param: `?deleted_at=is.null`
- URL param: `?is_active=is.true`
- URL param: `?is_verified=is.false`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL (NULL):** `WHERE deleted_at IS NULL`
- **SQL (boolean):** `WHERE is_active = 1` (SQLite uses 1/0 for boolean)
- **Notes:**
  - IS NULL: Direct mapping
  - Boolean: SQLite stores as INTEGER (0/1)
- **Implementation:** `SELECT * FROM users WHERE deleted_at IS NULL`

**Edge Cases:**
- For non-boolean columns, only `null` is meaningful
- `.eq(column, null)` is incorrect SQL (always returns NULL), use `.is()` instead
- `.is(column, true)` is equivalent to `.eq(column, true)` for boolean columns

**Related:** isDistinct(), eq(), not()

**Source:** PostgrestFilterBuilder.ts:307-327

---

## `isDistinct(column, value)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
isDistinct<ColumnName extends string>(
  column: ColumnName,
  value: Row[ColumnName]
): this
```

**Purpose:** Match rows where column IS DISTINCT FROM value. Unlike `neq()`, treats NULL as comparable value.

**Parameters:**
- `column` (string) - Column name to filter on
- `value` (any) - Value to compare against (including NULL)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Distinct from value (includes NULL)
const { data } = await client
  .from('users')
  .select()
  .isDistinct('name', 'John')
// Returns rows where name is NOT 'John' OR name IS NULL

// Distinct from NULL
const { data } = await client
  .from('users')
  .select()
  .isDistinct('deleted_at', null)
// Returns rows where deleted_at IS NOT NULL

// Comparison with neq()
// .neq('name', 'John') excludes NULLs
// .isDistinct('name', 'John') includes NULLs
```

**PostgREST Mapping:**
- URL param: `?name=isdistinct.John`
- URL param: `?deleted_at=isdistinct.null`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE name IS DISTINCT FROM ?`
- **Notes:**
  - SQLite 3.39+ supports `IS DISTINCT FROM`
  - For older SQLite: `WHERE (name != ? OR (name IS NULL AND ? IS NOT NULL) OR (name IS NOT NULL AND ? IS NULL))`
- **Implementation:** `SELECT * FROM users WHERE name IS DISTINCT FROM 'John'`
- **Workarounds:**
  - Manual NULL handling for SQLite < 3.39

**Edge Cases:**
- NULL IS DISTINCT FROM NULL: FALSE (they are equal)
- NULL IS DISTINCT FROM 'value': TRUE (they are different)
- 'value' IS DISTINCT FROM NULL: TRUE
- Two equal values: FALSE (they are not distinct)

**Related:** neq(), is(), not()

**Source:** PostgrestFilterBuilder.ts:329-349

---

## Summary

All 10 comparison filters have full SQLite compatibility with direct SQL mappings:

| Method | PostgREST | SQLite | Notes |
|--------|-----------|--------|-------|
| `eq()` | `eq.value` | `= ?` | Direct mapping |
| `neq()` | `neq.value` | `!= ?` | NULL handling differs |
| `gt()` | `gt.value` | `> ?` | Direct mapping |
| `gte()` | `gte.value` | `>= ?` | Direct mapping |
| `lt()` | `lt.value` | `< ?` | Direct mapping |
| `lte()` | `lte.value` | `<= ?` | Direct mapping |
| `in()` | `in.(...)` | `IN (?, ...)` | Direct mapping |
| `notIn()` | `not.in.(...)` | `NOT IN (?, ...)` | Direct mapping |
| `is()` | `is.null/true/false` | `IS NULL` / `= 1/0` | Boolean as INTEGER |
| `isDistinct()` | `isdistinct.value` | `IS DISTINCT FROM` | SQLite 3.39+ or manual |
