# Type Casting Reference (`::`)

**Category:** MEDIUM Priority | **Status:** ✅ Complete
**PostgREST docs:** https://docs.postgrest.org/en/v14/references/api/tables_views.html

---

## Overview

PostgREST supports PostgreSQL's `::` cast operator in **select columns only** (vertical filtering). Casting in WHERE clauses (horizontal filtering) is **not allowed** — use computed fields instead.

**Syntax:** `column::type`

```ts
// Supabase-JS
client.from('people').select('full_name,salary::text')

// PostgREST URL
GET /people?select=full_name,salary::text
```

Casting also works with:
- Aggregates on column: `order_details->tax_amount::numeric.sum()`
- Aggregate results: `amount.avg()::int`
- JSON paths: `metadata->price::numeric`
- Combined: `metadata->price::numeric.sum()::int`

---

## Restriction: Select Only

Casting is **prohibited in filters** to preserve index usage.

```
❌ /people?salary::text=eq.90000       -- NOT ALLOWED
✅ /people?select=salary::text          -- OK
```

**Workaround:** Create a computed field (PostgreSQL function on the table type) and filter on that instead.

---

## PostgreSQL → TypeScript → SQLite Mapping

Types recognized by `postgrest-js` select query parser (from `select-query-parser/types.ts`).

### Numeric Types

| PostgreSQL | Aliases | TS Type | SQLite | SQLite Cast |
|-----------|---------|---------|--------|-------------|
| `int2` | smallint | `number` | ✅ | `CAST(col AS INTEGER)` |
| `int4` | integer, int | `number` | ✅ | `CAST(col AS INTEGER)` |
| `int8` | bigint | `number` | ⚠️ | `CAST(col AS INTEGER)` — SQLite max 64-bit signed |
| `float4` | real | `number` | ✅ | `CAST(col AS REAL)` |
| `float8` | double precision | `number` | ✅ | `CAST(col AS REAL)` |
| `numeric` | decimal | `number` | ⚠️ | `CAST(col AS NUMERIC)` — no fixed precision in SQLite |

### String Types

| PostgreSQL | Aliases | TS Type | SQLite | SQLite Cast |
|-----------|---------|---------|--------|-------------|
| `text` | — | `string` | ✅ | `CAST(col AS TEXT)` |
| `varchar` | character varying | `string` | ✅ | `CAST(col AS TEXT)` — no length constraint |
| `bpchar` | char, character | `string` | ✅ | `CAST(col AS TEXT)` — no padding |
| `citext` | — | `string` | ⚠️ | `CAST(col AS TEXT)` — no case-insensitive type, use `COLLATE NOCASE` |

### Date/Time Types

| PostgreSQL | Aliases | TS Type | SQLite | SQLite Cast |
|-----------|---------|---------|--------|-------------|
| `date` | — | `string` | ⚠️ | `CAST(col AS TEXT)` — stored as ISO-8601 text |
| `time` | time without tz | `string` | ⚠️ | `CAST(col AS TEXT)` |
| `timetz` | time with tz | `string` | ⚠️ | `CAST(col AS TEXT)` — no native tz support |
| `timestamp` | timestamp without tz | `string` | ⚠️ | `CAST(col AS TEXT)` |
| `timestamptz` | timestamp with tz | `string` | ⚠️ | `CAST(col AS TEXT)` — no native tz support |

> SQLite has no native date/time types. Store as TEXT (ISO-8601) or INTEGER (Unix epoch). Use `date()`, `time()`, `datetime()` functions for manipulation.

### Boolean

| PostgreSQL | Aliases | TS Type | SQLite | SQLite Cast |
|-----------|---------|---------|--------|-------------|
| `bool` | boolean | `boolean` | ⚠️ | `CAST(col AS INTEGER)` — 0/1, no native bool |

### JSON Types

| PostgreSQL | Aliases | TS Type | SQLite | SQLite Cast |
|-----------|---------|---------|--------|-------------|
| `json` | — | `Json` | ⚠️ | `CAST(col AS TEXT)` — use json1 extension for ops |
| `jsonb` | — | `Json` | ⚠️ | `CAST(col AS TEXT)` — no binary JSON, use json1 |

### Binary

| PostgreSQL | Aliases | TS Type | SQLite | SQLite Cast |
|-----------|---------|---------|--------|-------------|
| `bytea` | — | `string` | ⚠️ | `CAST(col AS BLOB)` — different encoding (PG uses hex/escape, SQLite raw) |

### UUID

| PostgreSQL | Aliases | TS Type | SQLite | SQLite Cast |
|-----------|---------|---------|--------|-------------|
| `uuid` | — | `string` | ⚠️ | `CAST(col AS TEXT)` — no native UUID type, store as TEXT |

### Special Types

| PostgreSQL | Aliases | TS Type | SQLite | SQLite Cast |
|-----------|---------|---------|--------|-------------|
| `void` | — | `undefined` | N/A | N/A — no equivalent |
| `record` | — | `Record<string, unknown>` | ❌ | Not supported |
| `vector` | — | `string` | ❌ | Not supported — use sqlite-vss extension for vectors |

### Array Types (prefix `_`)

Any type above prefixed with `_` is its array form. Example: `_int4` = `integer[]`.

| PostgreSQL | TS Type | SQLite | Notes |
|-----------|---------|--------|-------|
| `_int2`, `_int4`, `_int8` | `number[]` | ❌ | No array type. Use JSON: `json_each()` |
| `_float4`, `_float8` | `number[]` | ❌ | Same — use JSON array |
| `_text`, `_varchar` | `string[]` | ❌ | Same — use JSON array |
| `_bool` | `boolean[]` | ❌ | Same — use JSON array |
| `_json`, `_jsonb` | `Json[]` | ❌ | Same — use JSON array |
| `_uuid` | `string[]` | ❌ | Same — use JSON array |
| `_timestamp`, `_timestamptz` | `string[]` | ❌ | Same — use JSON array |

---

## PostgreSQL Types NOT in postgrest-js Parser

These PG types exist but are **not recognized** by the postgrest-js TypeScript type system (they fall through to `unknown`). PostgREST itself still handles the cast server-side — only the client TS types are untyped.

| PostgreSQL | Category | SQLite |
|-----------|----------|--------|
| `money` | Monetary | ❌ No equivalent — use INTEGER (cents) or REAL |
| `inet`, `cidr` | Network | ❌ No equivalent — store as TEXT |
| `macaddr`, `macaddr8` | Network | ❌ No equivalent — store as TEXT |
| `point`, `line`, `lseg`, `box`, `path`, `polygon`, `circle` | Geometric | ❌ No equivalent |
| `tsvector`, `tsquery` | Full-text search | ❌ Use FTS5 extension instead |
| `xml` | XML | ❌ No equivalent — store as TEXT |
| `interval` | Date/time | ❌ No equivalent — store as TEXT or seconds |
| `bit`, `varbit` | Bit string | ❌ No equivalent |
| `pg_lsn`, `pg_snapshot` | System | ❌ N/A |

---

## SQLite Cast Summary

SQLite only has 5 storage classes. All casts map to one of:

| SQLite Type | Used For |
|-------------|----------|
| `INTEGER` | int2, int4, int8, bool (0/1) |
| `REAL` | float4, float8, numeric |
| `TEXT` | text, varchar, bpchar, citext, date, time, timestamp, uuid, json |
| `BLOB` | bytea |
| `NUMERIC` | numeric (affinity, not strict type) |

### Implementation Pattern

```ts
// Map PostgREST :: cast to SQLite CAST()
function mapCast(column: string, pgType: string): string {
  const sqliteType = {
    // Numeric
    int2: 'INTEGER', int4: 'INTEGER', int8: 'INTEGER',
    float4: 'REAL', float8: 'REAL', numeric: 'NUMERIC',
    // String
    text: 'TEXT', varchar: 'TEXT', bpchar: 'TEXT', citext: 'TEXT',
    // Date/time → TEXT (ISO-8601)
    date: 'TEXT', time: 'TEXT', timetz: 'TEXT',
    timestamp: 'TEXT', timestamptz: 'TEXT',
    // Other
    bool: 'INTEGER', json: 'TEXT', jsonb: 'TEXT',
    uuid: 'TEXT', bytea: 'BLOB',
  }[pgType]

  if (!sqliteType) throw new Error(`Unsupported cast: ::${pgType}`)
  return `CAST(${column} AS ${sqliteType})`
}
```

---

## Compatibility Matrix

| Status | Count | Types |
|--------|-------|-------|
| ✅ Full | 5 | int2, int4, float4, float8, text |
| ⚠️ Partial | 13 | int8, numeric, varchar, bpchar, citext, date, time, timetz, timestamp, timestamptz, bool, json/jsonb, uuid, bytea |
| ❌ None | 10+ | arrays, vector, record, money, inet, geometric, tsvector, xml, interval, bit |

**Partial** = cast works but semantics differ (no tz, no precision, no padding, etc.)
**None** = no SQLite equivalent type; requires workaround or extension

---

## Related

- `select()` in [core-operations.md](core-operations.md) — casting syntax
- [filters-specialized.md](filters-specialized.md) — `rpc()`, `schema()`
- [sqlite-compatibility-matrix.md](sqlite-compatibility-matrix.md) — full compatibility overview
- PostgREST computed fields — workaround for casting in filters
