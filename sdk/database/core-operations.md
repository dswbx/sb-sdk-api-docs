# Core Operations

**Category:** HIGH Priority | **Status:** ✅ Complete
**Methods:** 6 (from, select, insert, update, delete, upsert)

Core CRUD operations for interacting with tables and views.

---

## `from(relation)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
from<TableName extends string & keyof Schema['Tables']>(
  relation: TableName
): PostgrestQueryBuilder<Schema, Table, TableName>
```

**Purpose:** Start a query on a table or view.

**Parameters:**
- `relation` (string) - Table or view name. Must be non-empty string.

**Returns:** `PostgrestQueryBuilder` for chaining operations.

**Usage:**
```ts
// Query a table
const query = client.from('users')

// Query a view
const query = client.from('active_users_view')

// Chain with operations
const { data } = await client.from('users').select('*')
```

**PostgREST Mapping:**
- URL: `GET /users`
- Creates base URL for subsequent operations

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** Table/view reference in `FROM` clause
- **Notes:** Direct mapping. SQLite fully supports table and view references.
- **Implementation:** `SELECT * FROM users` or `SELECT * FROM active_users_view`

**Edge Cases:**
- Empty/whitespace string: Throws `Invalid relation name` error
- Non-string: Throws type error
- Non-existent table: Error on execution, not on call

**Related:** select(), insert(), update(), delete(), upsert()

**Source:** PostgrestClient.ts:78-101

---

## `select(columns, options)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
select<Query extends string = '*'>(
  columns?: Query,
  options?: {
    head?: boolean
    count?: 'exact' | 'planned' | 'estimated'
  }
): PostgrestFilterBuilder<Row, Result[], 'GET'>
```

**Purpose:** Fetch rows from table/view.

**Parameters:**
- `columns` (string, optional) - Comma-separated column list. Default: `'*'`. Supports:
  - Column selection: `'id,name,email'`
  - Column rename: `'id,name:display_name'`
  - Type casting: `'salary::text'` (see [type-casting-reference.md](type-casting-reference.md))
  - JSON traversal: `'metadata->theme'`
  - Related tables: `'id,posts(*)'`
- `options.head` (boolean, optional) - If true, only returns count, no data
- `options.count` (string, optional) - Count algorithm:
  - `'exact'` - Accurate COUNT(*), slower
  - `'planned'` - Uses Postgres stats, faster
  - `'estimated'` - Hybrid approach

**Returns:** `PostgrestFilterBuilder` with query result as `data` array.

**Usage:**
```ts
// All columns
const { data } = await client.from('users').select()

// Specific columns
const { data } = await client.from('users').select('id,name,email')

// With rename
const { data } = await client.from('users').select('id,name:userName')

// With count
const { data, count } = await client.from('users').select('*', { count: 'exact' })

// Head request (count only)
const { count } = await client.from('users').select('*', { head: true, count: 'exact' })

// JSON column
const { data } = await client.from('users').select('metadata->theme')

// Type casting (:: operator)
const { data } = await client.from('people').select('full_name,salary::text')

// Related tables
const { data } = await client.from('users').select('id,name,posts(*)')
```

**PostgREST Mapping:**
- HTTP method: GET (or HEAD if `head: true`)
- URL param: `?select=id,name,email`
- Headers:
  - `Prefer: count=exact` (if count specified)

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full (basic columns), ⚠️ Partial (related tables)
- **SQL:** `SELECT id, name, email FROM users`
- **Notes:**
  - Basic column selection: Direct mapping
  - Column rename: Use `AS` alias: `SELECT name AS userName`
  - Type casting: Use `CAST()`: `SELECT CAST(salary AS TEXT)` — see [type-casting-reference.md](type-casting-reference.md) for full PG→SQLite type mapping
  - JSON: Use `json_extract()`: `SELECT json_extract(metadata, '$.theme')`
  - Related tables: Requires JOIN logic, not automatic
  - Count: `SELECT COUNT(*) FROM users`
- **Workarounds:**
  - For related tables, manually construct JOINs
  - Implement custom parser for nested select syntax

**Edge Cases:**
- Empty string columns: Uses `'*'`
- Whitespace handling: Strips spaces except within quotes
- Invalid column: Error from database on execution
- JSON operators (->, ->>, #>, #>>): Require JSON support
- Type casting (`::`) only in select columns (vertical filtering), not in filter conditions (horizontal filtering). Use computed fields for filtered casting. Full reference: [type-casting-reference.md](type-casting-reference.md)

**Related:** insert(), update(), delete(), order(), limit(), eq()

**Source:** PostgrestQueryBuilder.ts:55-143

---

## `insert(values, options)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
insert<Row extends Relation['Insert']>(
  values: Row | Row[],
  options?: {
    count?: 'exact' | 'planned' | 'estimated'
    defaultToNull?: boolean
  }
): PostgrestFilterBuilder<Row, null, 'POST'>
```

**Purpose:** Insert new rows into table.

**Parameters:**
- `values` (object | array) - Single object or array of objects to insert
- `options.count` (string, optional) - Count algorithm (see select())
- `options.defaultToNull` (boolean, optional) - For bulk inserts:
  - `true` (default): Missing fields become NULL
  - `false`: Missing fields use column defaults

**Returns:** `PostgrestFilterBuilder` with `data: null` by default. Chain `.select()` to return inserted rows.

**Usage:**
```ts
// Single insert
const { error } = await client
  .from('users')
  .insert({ name: 'John', email: 'john@example.com' })

// Single insert with return
const { data, error } = await client
  .from('users')
  .insert({ name: 'John', email: 'john@example.com' })
  .select()

// Bulk insert
const { error } = await client
  .from('users')
  .insert([
    { name: 'John', email: 'john@example.com' },
    { name: 'Jane', email: 'jane@example.com' }
  ])

// With count
const { count } = await client
  .from('users')
  .insert({ name: 'John' }, { count: 'exact' })

// Use column defaults
const { data } = await client
  .from('users')
  .insert([
    { name: 'John' }, // email will use DEFAULT, not NULL
  ], { defaultToNull: false })
  .select()
```

**PostgREST Mapping:**
- HTTP method: POST
- URL param: `?columns="name","email"` (bulk insert only)
- Headers:
  - `Prefer: return=minimal` (default, no data returned)
  - `Prefer: count=exact` (if count specified)
  - `Prefer: missing=default` (if `defaultToNull: false`)
  - `Prefer: return=representation` (if `.select()` chained)
- Body: JSON object or array

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `INSERT INTO users (name, email) VALUES (?, ?)`
- **SQL (bulk):** `INSERT INTO users (name, email) VALUES (?, ?), (?, ?)`
- **Notes:**
  - Direct mapping to INSERT statement
  - defaultToNull: Use `DEFAULT` keyword or omit column
  - RETURNING clause (for select): `INSERT ... RETURNING *` (SQLite 3.35+)
- **Workarounds:**
  - For SQLite < 3.35, query after insert using last_insert_rowid()

**Edge Cases:**
- Empty object: Inserts row with all defaults (if allowed by schema)
- Missing required columns: Database constraint error
- Type mismatches: Database type error
- Array with inconsistent columns: All unique columns collected

**Related:** update(), upsert(), select()

**Source:** PostgrestQueryBuilder.ts:145-245

---

## `update(values, options)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
update<Row extends Relation['Update']>(
  values: Row,
  options?: {
    count?: 'exact' | 'planned' | 'estimated'
  }
): PostgrestFilterBuilder<Row, null, 'PATCH'>
```

**Purpose:** Modify existing rows in table.

**Parameters:**
- `values` (object) - Column-value pairs to update
- `options.count` (string, optional) - Count algorithm (see select())

**Returns:** `PostgrestFilterBuilder` with `data: null` by default. **Must chain filters** (`.eq()`, etc.) to target rows. Chain `.select()` to return updated rows.

**Usage:**
```ts
// Update with filter
const { error } = await client
  .from('users')
  .update({ name: 'Jane' })
  .eq('id', 1)

// Update with return
const { data, error } = await client
  .from('users')
  .update({ name: 'Jane' })
  .eq('id', 1)
  .select()

// Update multiple fields
const { error } = await client
  .from('users')
  .update({ name: 'Jane', email: 'jane@example.com' })
  .eq('status', 'inactive')

// With count
const { count } = await client
  .from('users')
  .update({ status: 'active' })
  .eq('verified', true)
  .select('*', { count: 'exact' })
```

**PostgREST Mapping:**
- HTTP method: PATCH
- URL params: Filters (e.g., `?id=eq.1`)
- Headers:
  - `Prefer: return=minimal` (default)
  - `Prefer: count=exact` (if count specified)
  - `Prefer: return=representation` (if `.select()` chained)
- Body: JSON object with update values

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `UPDATE users SET name = ?, email = ? WHERE id = ?`
- **Notes:**
  - Direct mapping to UPDATE statement
  - Filters map to WHERE clause
  - RETURNING clause: `UPDATE ... RETURNING *` (SQLite 3.35+)
- **Workarounds:**
  - For SQLite < 3.35, query after update

**Edge Cases:**
- No filters: Updates ALL rows (dangerous!)
- Empty values object: No-op, but still executes
- Non-existent columns: Database error
- No matching rows: Success with `count: 0`

**Related:** insert(), delete(), eq(), select()

**Source:** PostgrestQueryBuilder.ts:420-472

---

## `delete(options)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
delete(
  options?: {
    count?: 'exact' | 'planned' | 'estimated'
  }
): PostgrestFilterBuilder<Row, null, 'DELETE'>
```

**Purpose:** Remove rows from table.

**Parameters:**
- `options.count` (string, optional) - Count algorithm (see select())

**Returns:** `PostgrestFilterBuilder` with `data: null` by default. **Must chain filters** to target rows. Chain `.select()` to return deleted rows.

**Usage:**
```ts
// Delete with filter
const { error } = await client
  .from('users')
  .delete()
  .eq('id', 1)

// Delete with return
const { data, error } = await client
  .from('users')
  .delete()
  .eq('id', 1)
  .select()

// Delete multiple rows
const { error } = await client
  .from('users')
  .delete()
  .lt('last_login', '2020-01-01')

// With count
const { count } = await client
  .from('users')
  .delete({ count: 'exact' })
  .eq('status', 'deleted')
```

**PostgREST Mapping:**
- HTTP method: DELETE
- URL params: Filters (e.g., `?id=eq.1`)
- Headers:
  - `Prefer: return=minimal` (default)
  - `Prefer: count=exact` (if count specified)
  - `Prefer: return=representation` (if `.select()` chained)

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL:** `DELETE FROM users WHERE id = ?`
- **Notes:**
  - Direct mapping to DELETE statement
  - Filters map to WHERE clause
  - RETURNING clause: `DELETE ... RETURNING *` (SQLite 3.35+)
- **Workarounds:**
  - For SQLite < 3.35, select before delete if data needed

**Edge Cases:**
- No filters: Deletes ALL rows (dangerous!)
- No matching rows: Success with `count: 0`
- Foreign key constraints: May error if ON DELETE not configured

**Related:** update(), insert(), eq(), select()

**Source:** PostgrestQueryBuilder.ts:474-520

---

## `upsert(values, options)`

**Priority:** High | **SQLite:** ✅ Full

**Signature:**
```ts
upsert<Row extends Relation['Insert']>(
  values: Row | Row[],
  options?: {
    onConflict?: string
    ignoreDuplicates?: boolean
    count?: 'exact' | 'planned' | 'estimated'
    defaultToNull?: boolean
  }
): PostgrestFilterBuilder<Row, null, 'POST'>
```

**Purpose:** Insert new rows or update existing rows on conflict.

**Parameters:**
- `values` (object | array) - Single object or array to upsert
- `options.onConflict` (string, optional) - Comma-separated UNIQUE columns for conflict detection (e.g., `'email'` or `'id,email'`)
- `options.ignoreDuplicates` (boolean, optional):
  - `true`: Skip duplicates (INSERT IGNORE)
  - `false` (default): Merge/update duplicates
- `options.count` (string, optional) - Count algorithm (see select())
- `options.defaultToNull` (boolean, optional) - Same as insert()

**Returns:** `PostgrestFilterBuilder` with `data: null` by default. Chain `.select()` to return upserted rows.

**Usage:**
```ts
// Upsert single row (by primary key)
const { error } = await client
  .from('users')
  .upsert({ id: 1, name: 'John' })

// Upsert with custom conflict column
const { error } = await client
  .from('users')
  .upsert(
    { email: 'john@example.com', name: 'John' },
    { onConflict: 'email' }
  )

// Upsert bulk
const { error } = await client
  .from('users')
  .upsert([
    { id: 1, name: 'John' },
    { id: 2, name: 'Jane' }
  ])

// Ignore duplicates
const { error } = await client
  .from('users')
  .upsert(
    { email: 'john@example.com', name: 'John' },
    { onConflict: 'email', ignoreDuplicates: true }
  )

// With return
const { data } = await client
  .from('users')
  .upsert({ id: 1, name: 'Updated' })
  .select()
```

**PostgREST Mapping:**
- HTTP method: POST
- URL params:
  - `?on_conflict=email` (if specified)
  - `?columns="name","email"` (bulk only)
- Headers:
  - `Prefer: resolution=merge-duplicates` (default, update on conflict)
  - `Prefer: resolution=ignore-duplicates` (if `ignoreDuplicates: true`)
  - `Prefer: count=exact` (if count specified)
  - `Prefer: missing=default` (if `defaultToNull: false`)
  - `Prefer: return=representation` (if `.select()` chained)
- Body: JSON object or array

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **SQL (merge):** `INSERT INTO users (id, name) VALUES (?, ?) ON CONFLICT(id) DO UPDATE SET name = excluded.name`
- **SQL (ignore):** `INSERT OR IGNORE INTO users (id, name) VALUES (?, ?)`
- **Notes:**
  - SQLite supports ON CONFLICT since 3.24.0 (2018)
  - `onConflict`: Maps to ON CONFLICT(column)
  - `ignoreDuplicates: true`: Maps to INSERT OR IGNORE
  - `ignoreDuplicates: false`: Maps to ON CONFLICT DO UPDATE
  - RETURNING: Available in SQLite 3.35+
- **Workarounds:**
  - For older SQLite, use manual SELECT + INSERT/UPDATE logic

**Edge Cases:**
- No `onConflict`: Uses table's primary key by default
- Multiple conflict columns: `onConflict: 'col1,col2'`
- No unique constraint on `onConflict` columns: Database error
- `ignoreDuplicates: true`: Returns success even if row not inserted

**Related:** insert(), update(), select()

**Source:** PostgrestQueryBuilder.ts:247-418
