# Advanced Filters

**Category:** MEDIUM Priority | **Status:** ✅ Complete
**Methods:** 5 (textSearch, match, or, not, filter)

Advanced filtering capabilities: full-text search, logical operators, and custom filters.

---

## `textSearch(column, query, options)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
textSearch<ColumnName extends string & keyof Row>(
  column: ColumnName,
  query: string,
  options?: {
    config?: string
    type?: 'plain' | 'phrase' | 'websearch'
  }
): this
```

**Purpose:** Full-text search on text or tsvector columns.

**Parameters:**
- `column` (string) - Text or tsvector column to search
- `query` (string) - Search query text
- `options.config` (string, optional) - Text search configuration (e.g., 'english', 'spanish')
- `options.type` (string, optional) - Query type:
  - `'plain'`: Simple text matching (default)
  - `'phrase'`: Phrase search (exact order)
  - `'websearch'`: Web-style syntax (AND, OR, quotes, -)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Simple text search
const { data } = await client
  .from('posts')
  .select()
  .textSearch('content', 'typescript tutorial')

// Phrase search (exact order)
const { data } = await client
  .from('posts')
  .select()
  .textSearch('content', 'getting started', { type: 'phrase' })

// Web search syntax
const { data } = await client
  .from('posts')
  .select()
  .textSearch('content', 'typescript OR javascript -python', { type: 'websearch' })

// With language config
const { data } = await client
  .from('posts')
  .select()
  .textSearch('content', 'programación', { config: 'spanish' })

// Search on tsvector column
const { data } = await client
  .from('posts')
  .select()
  .textSearch('content_tsvector', 'javascript frameworks')
```

**PostgREST Mapping:**
- URL param: `?content=fts.typescript tutorial`
- URL param (phrase): `?content=phfts.getting started`
- URL param (websearch): `?content=wfts.typescript OR javascript -python`
- URL param (config): `?content=fts(english).query`
- Operators:
  - `fts`: plain text search
  - `phfts`: phrase search
  - `wfts`: websearch

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial (requires FTS5 extension)
- **SQL (FTS5):**
  ```sql
  -- Using FTS5 virtual table
  SELECT * FROM posts_fts
  WHERE posts_fts MATCH 'typescript tutorial'
  ```
- **Notes:**
  - Postgres uses built-in tsvector/tsquery
  - SQLite requires FTS5 extension (built-in since 3.9.0, 2015)
  - **Different syntax**: Postgres vs SQLite FTS
  - FTS5 table must be created separately:
    ```sql
    CREATE VIRTUAL TABLE posts_fts USING fts5(content, title);
    INSERT INTO posts_fts SELECT content, title FROM posts;
    ```
  - Query syntax differs:
    - Postgres: `to_tsquery('typescript & tutorial')`
    - SQLite FTS5: `MATCH 'typescript tutorial'` (implicit AND)
- **Workarounds:**
  - Create FTS5 virtual table mirroring main table
  - Map PostgREST search syntax to FTS5 syntax:
    - plain: `MATCH 'term1 term2'` (AND by default)
    - phrase: `MATCH '"exact phrase"'`
    - websearch: Parse and convert to FTS5 syntax (AND, OR, NOT)
  - Maintain FTS index with triggers
  - Language config: FTS5 uses tokenizers, not configs

**Edge Cases:**
- Empty query: May return all or none (database-dependent)
- Special characters: Requires escaping
- Stop words: Filtered by text search config
- tsvector vs text: tsvector is preprocessed, faster

**Related:** like(), ilike()

**Source:** PostgrestFilterBuilder.ts:559-595

---

## `match(query)`

**Priority:** Medium | **SQLite:** ✅ Full

**Signature:**
```ts
match<ColumnName extends string & keyof Row>(
  query: Record<ColumnName, Row[ColumnName]>
): this
```

**Purpose:** Match multiple columns with equality. Shorthand for multiple `eq()` calls.

**Parameters:**
- `query` (object) - Key-value pairs: column names to match values

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Multiple equality filters
const { data } = await client
  .from('users')
  .select()
  .match({ status: 'active', role: 'admin' })
// Equivalent to: .eq('status', 'active').eq('role', 'admin')

// Single match
const { data } = await client
  .from('products')
  .select()
  .match({ category: 'electronics' })

// Complex example
const filters = { status: 'published', author_id: 123, featured: true }
const { data } = await client
  .from('posts')
  .select()
  .match(filters)
```

**PostgREST Mapping:**
- URL params: `?status=eq.active&role=eq.admin`
- Expands object to multiple `eq.` filters

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `WHERE status = ? AND role = ?`
- **Notes:**
  - Direct mapping to multiple AND conditions
  - Each key-value pair becomes `column = value`
- **Implementation:**
  ```sql
  SELECT * FROM users
  WHERE status = 'active'
    AND role = 'admin'
  ```

**Edge Cases:**
- Empty object: No filters (returns all)
- NULL values: Should use `.is()` for NULL checks
- Order doesn't matter (all ANDed together)

**Related:** eq(), and, or()

**Source:** PostgrestFilterBuilder.ts:597-611

---

## `or(filters, options)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
or(
  filters: string,
  options?: {
    referencedTable?: string
  }
): this
```

**Purpose:** Match rows satisfying at least one filter condition. Logical OR.

**Parameters:**
- `filters` (string) - PostgREST filter syntax string (must be manually constructed)
- `options.referencedTable` (string, optional) - Apply OR to related table

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Basic OR
const { data } = await client
  .from('users')
  .select()
  .or('status.eq.active,status.eq.pending')
// WHERE status = 'active' OR status = 'pending'

// Different columns
const { data } = await client
  .from('users')
  .select()
  .or('role.eq.admin,role.eq.moderator,is_staff.eq.true')

// Combining with AND (outer filter)
const { data } = await client
  .from('users')
  .select()
  .eq('is_verified', true)
  .or('status.eq.active,status.eq.pending')
// WHERE is_verified = true AND (status = 'active' OR status = 'pending')

// On related table
const { data } = await client
  .from('users')
  .select('*, posts(*)')
  .or('status.eq.published,featured.eq.true', { referencedTable: 'posts' })
```

**PostgREST Mapping:**
- URL param: `?or=(status.eq.active,status.eq.pending)`
- URL param (related): `?posts.or=(status.eq.published,featured.eq.true)`
- **Important**: Filters string must follow exact PostgREST syntax

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial (requires parsing PostgREST syntax)
- **SQL:** `WHERE (status = 'active' OR status = 'pending')`
- **Notes:**
  - Must parse PostgREST filter string: `status.eq.active,status.eq.pending`
  - Split by `,` for OR conditions
  - Parse each condition: `column.operator.value`
  - Map operators to SQL: `eq` → `=`, `gt` → `>`, etc.
- **Implementation:**
  ```sql
  SELECT * FROM users
  WHERE (status = 'active' OR status = 'pending')
  ```
- **Workarounds:**
  - Build PostgREST filter string parser
  - Map operator strings to SQL operators
  - Parenthesize OR groups properly

**Edge Cases:**
- **Manual syntax**: Requires constructing PostgREST filter string
- Invalid syntax: Runtime error
- Cannot OR across tables (use referencedTable for related tables)
- Needs proper escaping for special characters

**Related:** not(), match(), eq()

**Source:** PostgrestFilterBuilder.ts:637-662

---

## `not(column, operator, value)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
not<ColumnName extends string & keyof Row>(
  column: ColumnName,
  operator: FilterOperator,
  value: Row[ColumnName]
): this
```

**Purpose:** Negate a filter condition. Match rows NOT satisfying the filter.

**Parameters:**
- `column` (string) - Column name to filter on
- `operator` (string) - PostgREST operator to negate (e.g., 'eq', 'gt', 'like', 'in', 'is')
- `value` (any) - Value for the operator (must follow PostgREST syntax)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// NOT equal
const { data } = await client
  .from('users')
  .select()
  .not('status', 'eq', 'banned')
// WHERE NOT (status = 'banned')

// NOT in
const { data } = await client
  .from('products')
  .select()
  .not('category', 'in', '(electronics,books)')
// WHERE NOT (category IN ('electronics', 'books'))

// NOT NULL
const { data } = await client
  .from('users')
  .select()
  .not('deleted_at', 'is', 'null')
// WHERE deleted_at IS NOT NULL

// NOT like
const { data } = await client
  .from('users')
  .select()
  .not('email', 'like', '%@test.com')
// WHERE NOT (email LIKE '%@test.com')
```

**PostgREST Mapping:**
- URL param: `?status=not.eq.banned`
- URL param: `?category=not.in.(electronics,books)`
- URL param: `?deleted_at=not.is.null`
- Format: `?column=not.operator.value`

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial (requires operator mapping)
- **SQL:** `WHERE NOT (status = ?)`
- **Notes:**
  - Must parse operator string and map to SQL
  - Wrap condition in NOT()
  - Operator value must follow PostgREST format
- **Implementation:**
  ```sql
  -- not.eq.banned
  SELECT * FROM users WHERE NOT (status = 'banned')

  -- not.in.(electronics,books)
  SELECT * FROM products WHERE NOT (category IN ('electronics', 'books'))

  -- not.is.null
  SELECT * FROM users WHERE deleted_at IS NOT NULL
  ```
- **Workarounds:**
  - Map PostgREST operators to SQL
  - Parse `in.(...)` format for arrays
  - Handle special operators like `is`, `like`

**Edge Cases:**
- **Manual syntax**: operator and value must follow PostgREST format
- Double negative: `.not('col', 'neq', 'value')` → NOT (col != value) → col = value
- NULL handling: `.not('col', 'is', null)` for IS NOT NULL
- Use `.notIn()` shortcut instead of `.not('col', 'in', '(...)')`

**Related:** eq(), neq(), or()

**Source:** PostgrestFilterBuilder.ts:613-635

---

## `filter(column, operator, value)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
filter<ColumnName extends string & keyof Row>(
  column: ColumnName,
  operator: string,
  value: unknown
): this
```

**Purpose:** Escape hatch for custom filters using raw PostgREST operators.

**Parameters:**
- `column` (string) - Column name to filter on
- `operator` (string) - PostgREST operator string (e.g., 'eq', 'gt', 'not.in', 'cs')
- `value` (unknown) - Value for the operator (must follow PostgREST format)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Use when specific method not available
const { data } = await client
  .from('products')
  .select()
  .filter('price', 'gte', 100)
// Equivalent to .gte('price', 100)

// Complex operator
const { data } = await client
  .from('posts')
  .select()
  .filter('tags', 'cs', '{javascript,typescript}')
// Contains operator for arrays

// Not in
const { data } = await client
  .from('users')
  .select()
  .filter('status', 'not.in', '(banned,deleted)')
```

**PostgREST Mapping:**
- URL param: `?price=gte.100`
- URL param: `?tags=cs.{javascript,typescript}`
- Format: `?column=operator.value`
- **Direct passthrough**: operator and value used as-is

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial (requires full operator mapping)
- **SQL:** Depends on operator
- **Notes:**
  - Must implement PostgREST operator → SQL mapping
  - Same as implementing all other filter methods
  - No validation of operator or value format
- **Workarounds:**
  - Full operator parser (same as `or()` and `not()`)
  - Map all PostgREST operators:
    - `eq` → `=`
    - `neq` → `!=`
    - `gt`, `gte`, `lt`, `lte` → `>`, `>=`, `<`, `<=`
    - `like`, `ilike` → `LIKE` (with COLLATE)
    - `in`, `not.in` → `IN`, `NOT IN`
    - `is`, `isdistinct` → `IS`, `IS DISTINCT FROM`
    - `cs`, `cd`, `ov` → JSON operations
    - `fts`, `phfts`, `wfts` → FTS5 MATCH

**Edge Cases:**
- **No validation**: Incorrect operator/value format causes runtime error
- Use specific methods (`.eq()`, `.gt()`, etc.) when available
- Escape hatch for missing methods or custom extensions
- Requires understanding PostgREST operator syntax

**Related:** or(), not(), all filter methods

**Source:** PostgrestFilterBuilder.ts:664-686

---

## Summary

Advanced filters with SQLite compatibility:

| Method | PostgREST | SQLite | Complexity |
|--------|-----------|--------|-----------|
| `textSearch()` | `fts/phfts/wfts.query` | FTS5 `MATCH` | ⚠️ Requires FTS5, syntax differs |
| `match()` | Multiple `eq.` | Multiple `AND =` | ✅ Direct mapping |
| `or()` | `or=(...)` | `OR` clause | ⚠️ Requires syntax parser |
| `not()` | `not.op.val` | `NOT (...)` | ⚠️ Requires operator mapping |
| `filter()` | `op.val` | Depends | ⚠️ Requires full operator mapping |

**Implementation Priority for SQLite:**
1. `match()` - Simple, direct mapping
2. `textSearch()` - Requires FTS5 setup
3. `or()`, `not()`, `filter()` - Requires PostgREST syntax parser

**Parser Requirements:**
- Operator string → SQL operator mapping
- Value format parsing (arrays, ranges, etc.)
- Proper escaping and quoting
