# Specialized Filters & Methods (Brief)

**Category:** LOW Priority | **Status:** ✅ Complete
**Methods:** 26+ specialized/control methods

Less commonly used filters, control methods, and type utilities.

---

## Regex Filters

### `regexMatch(column, pattern)`

**Priority:** Low | **SQLite:** ⚠️ Partial

**Signature:** `regexMatch(column: string, pattern: string): this`

**Purpose:** Case-sensitive regex match using PostgreSQL `~` operator.

**Usage:** `client.from('users').select().regexMatch('email', '^[a-z]+@company\\.com$')`

**SQLite:** ⚠️ Requires loading REGEXP function (not built-in). Use LIKE for simple patterns.

**Source:** PostgrestFilterBuilder.ts:279-291

---

### `regexIMatch(column, pattern)`

**Priority:** Low | **SQLite:** ⚠️ Partial

**Signature:** `regexIMatch(column: string, pattern: string): this`

**Purpose:** Case-insensitive regex match using PostgreSQL `~*` operator.

**Usage:** `client.from('users').select().regexIMatch('name', '^john.*')`

**SQLite:** ⚠️ Requires loading REGEXP function + case-insensitive flag.

**Source:** PostgrestFilterBuilder.ts:293-305

---

## Control Methods

### `rollback()`

**Priority:** Low | **SQLite:** ❌ No

**Signature:** `rollback(): this`

**Purpose:** Rollback query transaction. Data still returned but not committed. **PostgREST-specific.**

**Usage:** `client.from('users').insert({ name: 'Test' }).rollback()` (for testing)

**PostgREST:** `Prefer: tx=rollback` header

**SQLite:** ❌ Not applicable. PostgREST feature for dry-run testing. In SQLite, use explicit transactions:
```ts
db.exec('BEGIN TRANSACTION')
db.exec('INSERT ...')
db.exec('ROLLBACK')
```

**Source:** PostgrestTransformBuilder.ts:317-325

---

### `maxAffected(value)`

**Priority:** Low | **SQLite:** ✅ Full

**Signature:** `maxAffected(value: number): this`

**Purpose:** Set max rows that can be affected by UPDATE/DELETE. Error if exceeded.

**Usage:** `client.from('users').update({ status: 'deleted' }).eq('old', true).maxAffected(100)`

**PostgREST:** `Prefer: handling=strict, max-affected=100` (requires PostgREST 13+)

**SQLite:** ✅ Can implement by checking `changes()` after query:
```ts
db.run('UPDATE users SET status = ? WHERE old = ?', ['deleted', true])
if (db.changes() > maxAffected) {
  db.exec('ROLLBACK')
  throw new Error(`Max affected exceeded: ${db.changes()} > ${maxAffected}`)
}
```

**Source:** PostgrestTransformBuilder.ts:354-372

---

### `returns<NewResult>()`

**Priority:** Low | **SQLite:** N/A

**Signature:** `returns<NewResult>(): this`

**Purpose:** Override TypeScript return type. **Deprecated.** Use `overrideTypes<>()` instead.

**Usage:** `client.from('users').select().returns<CustomType[]>()`

**SQLite:** N/A - TypeScript-only type casting.

**Source:** PostgrestTransformBuilder.ts:327-351, PostgrestBuilder.ts:268-285

---

### `overrideTypes<NewResult, Options>()`

**Priority:** Low | **SQLite:** N/A

**Signature:** `overrideTypes<NewResult, Options extends { merge?: boolean }>(): this`

**Purpose:** Override TypeScript return type with merge option.

**Usage:**
```ts
// Merge with existing types (default)
client.from('users').select().overrideTypes<{ custom: string }>()

// Replace types completely
client.from('users').select().overrideTypes<CustomType[], { merge: false }>()
```

**SQLite:** N/A - TypeScript-only type casting.

**Source:** PostgrestBuilder.ts:287-333

---

## Schema & RPC

### `schema(schemaName)`

**Priority:** Medium/Low | **SQLite:** ❌ No

**Signature:** `schema<Schema extends string>(schema: Schema): PostgrestClient`

**Purpose:** Switch to different PostgreSQL schema.

**Usage:** `client.schema('private').from('users').select()`

**PostgREST:** `Accept-Profile: private` / `Content-Profile: private` headers

**SQLite:** ❌ No schema support. Use separate databases with ATTACH:
```sql
ATTACH DATABASE 'other.db' AS private;
SELECT * FROM private.users;
```

**Source:** PostgrestClient.ts:104-123

---

### `rpc(fn, args, options)`

**Priority:** High/Medium | **SQLite:** ⚠️ Partial

**Signature:** `rpc<FnName>(fn: FnName, args?: Args, options?: { head?: boolean, get?: boolean, count?: string }): PostgrestFilterBuilder`

**Purpose:** Call PostgreSQL stored procedure/function.

**Usage:**
```ts
// Simple call
const { data } = await client.rpc('get_user_summary', { user_id: 123 })

// With filters
const { data } = await client
  .rpc('search_products', { keyword: 'phone' })
  .eq('category', 'electronics')
  .limit(10)
```

**PostgREST:** `POST /rpc/function_name` with args as body

**SQLite:** ⚠️ No stored procedures. Workarounds:
- Define functions in application code
- Use triggers for some logic
- Create views for complex queries

**Source:** PostgrestClient.ts:125-227

---

## Additional Methods

The following methods are covered in other documentation:

- **select() after mutation** - PostgrestTransformBuilder.ts:18-73 (see transforms.md)
- **count options** - Used in select(), insert(), update(), delete(), upsert() (see core-operations.md)
- **Prefer headers** - Various (return=representation, count=exact, etc.)
- **Type utilities** - TypeScript generics, type resolution (N/A for SQLite)

---

## Summary

LOW priority methods by SQLite compatibility:

| Method | SQLite | Notes |
|--------|--------|-------|
| `regexMatch()` | ⚠️ Partial | Requires REGEXP extension |
| `regexIMatch()` | ⚠️ Partial | Requires REGEXP extension |
| `rangeGt/Gte/Lt/Lte/Adjacent()` | ❌ No | No range types, use separate columns |
| `rollback()` | ❌ No | PostgREST-specific, use explicit transactions |
| `maxAffected()` | ✅ Full | Check changes() after query |
| `returns()` / `overrideTypes()` | N/A | TypeScript only |
| `schema()` | ❌ No | Use ATTACH DATABASE |
| `rpc()` | ⚠️ Partial | No stored procs, use app functions |

**Implementation Priority:**
- Skip range operators if targeting SQLite
- Implement maxAffected() with changes() check
- Map rpc() to application functions
- Add REGEXP extension if regex needed
