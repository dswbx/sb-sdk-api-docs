# Pattern Matching Filters

**Category:** MEDIUM Priority | **Status:** ✅ Complete
**Methods:** 6 (like, ilike, likeAllOf, likeAnyOf, ilikeAllOf, ilikeAnyOf)

SQL LIKE pattern matching filters for text searching.

---

## `like(column, pattern)`

**Priority:** Medium | **SQLite:** ✅ Full

**Signature:**
```ts
like<ColumnName extends string & keyof Row>(
  column: ColumnName,
  pattern: string
): this
```

**Purpose:** Match rows where column matches SQL LIKE pattern (case-sensitive).

**Parameters:**
- `column` (string) - Column name to filter on
- `pattern` (string) - SQL LIKE pattern:
  - `%`: Matches any sequence of characters
  - `_`: Matches any single character
  - Literal: Use backslash to escape: `\%`, `\_`

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Starts with
const { data } = await client
  .from('users')
  .select()
  .like('name', 'John%')

// Ends with
const { data } = await client
  .from('users')
  .select()
  .like('email', '%@gmail.com')

// Contains
const { data } = await client
  .from('products')
  .select()
  .like('name', '%phone%')

// Single character wildcard
const { data } = await client
  .from('codes')
  .select()
  .like('code', 'A_C') // Matches 'ABC', 'A1C', etc.

// Escape special chars
const { data } = await client
  .from('users')
  .select()
  .like('bio', '%100\\%%') // Matches "100%" literally
```

**PostgREST Mapping:**
- URL param: `?name=like.John%`
- URL param: `?email=like.*@gmail.com`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE name LIKE ?`
- **Notes:**
  - Direct mapping to LIKE operator
  - SQLite LIKE is case-insensitive by default for ASCII (A-Z)
  - For case-sensitive: Use `COLLATE BINARY`: `name LIKE ? COLLATE BINARY`
- **Implementation:** `SELECT * FROM users WHERE name LIKE 'John%' COLLATE BINARY`

**Edge Cases:**
- Case sensitivity: Case-sensitive match. Use `.ilike()` for case-insensitive.
- Empty pattern: Matches empty strings
- No wildcards: Same as `.eq()`
- Unicode: Behavior may vary between Postgres and SQLite

**Related:** ilike(), likeAllOf(), likeAnyOf()

**Source:** PostgrestFilterBuilder.ts:189-200

---

## `ilike(column, pattern)`

**Priority:** Medium | **SQLite:** ✅ Full

**Signature:**
```ts
ilike<ColumnName extends string & keyof Row>(
  column: ColumnName,
  pattern: string
): this
```

**Purpose:** Match rows where column matches SQL LIKE pattern (case-insensitive).

**Parameters:**
- `column` (string) - Column name to filter on
- `pattern` (string) - SQL LIKE pattern (same as `like()`)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Case-insensitive search
const { data } = await client
  .from('users')
  .select()
  .ilike('name', 'john%')
// Matches 'John', 'JOHN', 'john', 'Johnny', etc.

// Case-insensitive contains
const { data } = await client
  .from('products')
  .select()
  .ilike('description', '%iphone%')
// Matches 'iPhone', 'IPHONE', 'iphone', etc.
```

**PostgREST Mapping:**
- URL param: `?name=ilike.john%`

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE name LIKE ? COLLATE NOCASE`
- **Notes:**
  - Use COLLATE NOCASE for case-insensitive
  - SQLite LIKE is case-insensitive by default for ASCII, but COLLATE NOCASE is explicit
- **Implementation:** `SELECT * FROM users WHERE name LIKE 'john%' COLLATE NOCASE`

**Edge Cases:**
- Same as `like()` but case-insensitive
- Common for user searches, email lookups

**Related:** like(), ilikeAllOf(), ilikeAnyOf()

**Source:** PostgrestFilterBuilder.ts:234-245

---

## `likeAllOf(column, patterns)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
likeAllOf<ColumnName extends string & keyof Row>(
  column: ColumnName,
  patterns: readonly string[]
): this
```

**Purpose:** Match rows where column matches ALL patterns (case-sensitive).

**Parameters:**
- `column` (string) - Column name to filter on
- `patterns` (string[]) - Array of SQL LIKE patterns (must match ALL)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Must contain both keywords
const { data } = await client
  .from('products')
  .select()
  .likeAllOf('description', ['%phone%', '%camera%'])
// Matches: "Smartphone with camera", "Camera phone"
// Doesn't match: "Phone without camera feature", "Camera only"

// Multiple constraints
const { data } = await client
  .from('users')
  .select()
  .likeAllOf('bio', ['%engineer%', '%python%', '%react%'])
```

**PostgREST Mapping:**
- URL param: `?description=like(all).{%phone%,%camera%}`

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial (requires multiple conditions)
- **SQL:** `WHERE name LIKE ? AND name LIKE ? AND name LIKE ? ...`
- **Notes:**
  - No native "LIKE ALL" operator
  - Must expand to multiple AND conditions
- **Implementation:**
  ```sql
  SELECT * FROM products
  WHERE description LIKE '%phone%' COLLATE BINARY
    AND description LIKE '%camera%' COLLATE BINARY
  ```

**Edge Cases:**
- Empty array: No filter (matches all)
- Single pattern: Same as `.like()`
- Order doesn't matter (all must match)

**Related:** likeAnyOf(), ilikeAllOf(), like()

**Source:** PostgrestFilterBuilder.ts:202-216

---

## `likeAnyOf(column, patterns)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
likeAnyOf<ColumnName extends string & keyof Row>(
  column: ColumnName,
  patterns: readonly string[]
): this
```

**Purpose:** Match rows where column matches ANY pattern (case-sensitive).

**Parameters:**
- `column` (string) - Column name to filter on
- `patterns` (string[]) - Array of SQL LIKE patterns (match at least one)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Match either pattern
const { data } = await client
  .from('users')
  .select()
  .likeAnyOf('email', ['%@gmail.com', '%@yahoo.com'])

// Multiple alternatives
const { data } = await client
  .from('products')
  .select()
  .likeAnyOf('name', ['%phone%', '%tablet%', '%laptop%'])
```

**PostgREST Mapping:**
- URL param: `?email=like(any).{%@gmail.com,%@yahoo.com}`

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial (requires multiple conditions)
- **SQL:** `WHERE name LIKE ? OR name LIKE ? OR name LIKE ? ...`
- **Notes:**
  - No native "LIKE ANY" operator
  - Must expand to multiple OR conditions
  - Consider wrapping in subquery or using UNION
- **Implementation:**
  ```sql
  SELECT * FROM users
  WHERE email LIKE '%@gmail.com' COLLATE BINARY
     OR email LIKE '%@yahoo.com' COLLATE BINARY
  ```

**Edge Cases:**
- Empty array: No filter (matches all)
- Single pattern: Same as `.like()`
- Order doesn't matter

**Related:** likeAllOf(), ilikeAnyOf(), like()

**Source:** PostgrestFilterBuilder.ts:218-232

---

## `ilikeAllOf(column, patterns)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
ilikeAllOf<ColumnName extends string & keyof Row>(
  column: ColumnName,
  patterns: readonly string[]
): this
```

**Purpose:** Match rows where column matches ALL patterns (case-insensitive).

**Parameters:**
- `column` (string) - Column name to filter on
- `patterns` (string[]) - Array of SQL LIKE patterns (must match ALL)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Case-insensitive, must match all
const { data } = await client
  .from('products')
  .select()
  .ilikeAllOf('description', ['%PHONE%', '%camera%'])
// Matches: "iPhone with Camera", "phone with camera"
```

**PostgREST Mapping:**
- URL param: `?description=ilike(all).{%PHONE%,%camera%}`

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial
- **SQL:** `WHERE name LIKE ? COLLATE NOCASE AND name LIKE ? COLLATE NOCASE ...`
- **Implementation:**
  ```sql
  SELECT * FROM products
  WHERE description LIKE '%PHONE%' COLLATE NOCASE
    AND description LIKE '%camera%' COLLATE NOCASE
  ```

**Related:** ilikeAnyOf(), likeAllOf(), ilike()

**Source:** PostgrestFilterBuilder.ts:247-261

---

## `ilikeAnyOf(column, patterns)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
ilikeAnyOf<ColumnName extends string & keyof Row>(
  column: ColumnName,
  patterns: readonly string[]
): this
```

**Purpose:** Match rows where column matches ANY pattern (case-insensitive).

**Parameters:**
- `column` (string) - Column name to filter on
- `patterns` (string[]) - Array of SQL LIKE patterns (match at least one)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Case-insensitive, match any
const { data } = await client
  .from('users')
  .select()
  .ilikeAnyOf('email', ['%@GMAIL.com', '%@yahoo.COM'])
// Matches: '@gmail.com', '@Gmail.com', '@YAHOO.com', etc.
```

**PostgREST Mapping:**
- URL param: `?email=ilike(any).{%@GMAIL.com,%@yahoo.COM}`

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial
- **SQL:** `WHERE name LIKE ? COLLATE NOCASE OR name LIKE ? COLLATE NOCASE ...`
- **Implementation:**
  ```sql
  SELECT * FROM users
  WHERE email LIKE '%@GMAIL.com' COLLATE NOCASE
     OR email LIKE '%@yahoo.COM' COLLATE NOCASE
  ```

**Related:** ilikeAllOf(), likeAnyOf(), ilike()

**Source:** PostgrestFilterBuilder.ts:263-277

---

## Summary

Pattern matching filters with SQLite compatibility:

| Method | PostgREST | SQLite | Notes |
|--------|-----------|--------|-------|
| `like()` | `like.pattern` | `LIKE ? COLLATE BINARY` | Full support |
| `ilike()` | `ilike.pattern` | `LIKE ? COLLATE NOCASE` | Full support |
| `likeAllOf()` | `like(all).{...}` | Multiple `AND LIKE` | Expand to AND chain |
| `likeAnyOf()` | `like(any).{...}` | Multiple `OR LIKE` | Expand to OR chain |
| `ilikeAllOf()` | `ilike(all).{...}` | Multiple `AND LIKE NOCASE` | Expand to AND chain |
| `ilikeAnyOf()` | `ilike(any).{...}` | Multiple `OR LIKE NOCASE` | Expand to OR chain |

**SQLite Implementation Notes:**
- Basic `like/ilike`: Direct mapping with COLLATE
- `*AllOf/*AnyOf`: Expand arrays to multiple conditions
- Wildcards: `%` and `_` work the same in SQLite and Postgres
