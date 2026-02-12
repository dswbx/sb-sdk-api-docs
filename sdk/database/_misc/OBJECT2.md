# Database API Object Representation v2

Simplified unified JSON format covering all 66 Database API methods.

---

## Changes from v1

- Removed `range` - use `limit`/`offset`
- Removed `abortSignal` - runtime concern, not serializable
- Removed `format` - transport layer determines response format
- Merged `$is` into `$eq`/`$neq` - parser handles `$eq: null` → `IS NULL`
- Unified `single`/`maybeSingle` into `cardinality: "one" | "maybe" | "many"`
- JSON access uses JSONPath (`$.path.to.key`) instead of SQL operators
- `order` is always array format with consistent structure
- Renamed `with` → `embed` with join type, hint, spread support

---

## Core Structure

```json
{
  "type": "query",
  "from": "table",
  "schema": "public",
  "select": [...],
  "embed": {...},
  "where": {...},
  "order": [...],
  "limit": 10,
  "offset": 0,
  "$meta": {...}
}
```

**Operation types:** `query` (default), `insert`, `update`, `delete`, `upsert`, `rpc`

---

## 1. Query (SELECT)

```json
{
  "from": "products",
  "select": [
    "id",
    "name",
    "price",
    { "description": { "as": "desc" } },
    { "metadata": { "path": "$.theme.color", "as": "color" } },
    { "tags": { "path": "$[0]", "as": "first_tag" } }
  ],
  "embed": {
    "categories": {
      "as": "cats",
      "select": ["id", "name"],
      "where": { "active": { "$eq": true } },
      "limit": 5
    },
    "reviews": {
      "join": "inner",
      "select": ["rating", "comment"],
      "order": [{ "column": "created_at", "direction": "desc" }]
    }
  },
  "where": {
    "status": { "$eq": "active" },
    "price": { "$gt": 100, "$lt": 500 },
    "deleted_at": { "$eq": null }
  },
  "order": [
    { "column": "price", "direction": "asc" },
    { "column": "name", "direction": "desc", "nullsFirst": true }
  ],
  "limit": 50,
  "offset": 0,
  "$meta": {
    "cardinality": "many",
    "count": "exact"
  }
}
```

---

## 2. Insert

```json
{
  "type": "insert",
  "from": "products",
  "values": [
    { "name": "iPhone 15", "price": 999 },
    { "name": "Galaxy S24", "price": 899 }
  ],
  "select": ["id", "name", "created_at"],
  "$meta": {
    "defaultsToNull": true,
    "count": "exact"
  }
}
```

---

## 3. Update

```json
{
  "type": "update",
  "from": "products",
  "values": {
    "price": 799,
    "updated_at": "2026-02-02T00:00:00Z"
  },
  "where": {
    "id": { "$eq": 123 }
  },
  "select": ["id", "price", "updated_at"],
  "$meta": {
    "maxAffected": 1
  }
}
```

---

## 4. Delete

```json
{
  "type": "delete",
  "from": "products",
  "where": {
    "id": { "$in": [1, 2, 3] }
  },
  "select": ["id", "name"],
  "$meta": {
    "maxAffected": 10
  }
}
```

---

## 5. Upsert

```json
{
  "type": "upsert",
  "from": "inventory",
  "values": [
    { "product_id": 1, "quantity": 50 },
    { "product_id": 2, "quantity": 30 }
  ],
  "onConflict": "product_id",
  "ignoreDuplicates": false,
  "select": ["product_id", "quantity", "updated_at"]
}
```

---

## 6. RPC

```json
{
  "type": "rpc",
  "function": "calculate_discount",
  "args": {
    "product_id": 123,
    "discount_percent": 15
  },
  "select": ["final_price", "tax_amount"],
  "where": {
    "status": { "$eq": "success" }
  },
  "order": [{ "column": "final_price", "direction": "asc" }],
  "limit": 10
}
```

---

## Operators

### Comparison
| Operator | SQL | Notes |
|----------|-----|-------|
| `$eq` | `=` / `IS NULL` | `$eq: null` → `IS NULL` |
| `$neq` | `<>` / `IS NOT NULL` | `$neq: null` → `IS NOT NULL` |
| `$gt` | `>` | |
| `$gte` | `>=` | |
| `$lt` | `<` | |
| `$lte` | `<=` | |
| `$in` | `IN (...)` | |
| `$notIn` | `NOT IN (...)` | |
| `$isDistinct` | `IS DISTINCT FROM` | NULL-safe inequality |

### Pattern Matching
| Operator | SQL |
|----------|-----|
| `$like` | `LIKE` |
| `$ilike` | `ILIKE` |
| `$likeAllOf` | `LIKE ALL(...)` |
| `$likeAnyOf` | `LIKE ANY(...)` |
| `$ilikeAllOf` | `ILIKE ALL(...)` |
| `$ilikeAnyOf` | `ILIKE ANY(...)` |
| `$regex` | `~` |
| `$iregex` | `~*` |

### Array/JSON
| Operator | SQL | Notes |
|----------|-----|-------|
| `$contains` | `@>` | Array/JSON contains |
| `$containedBy` | `<@` | Contained by |
| `$overlaps` | `&&` | Has common elements |

### Range (PostgreSQL only)
| Operator | SQL |
|----------|-----|
| `$rangeGt` | `>>` |
| `$rangeGte` | `&>` |
| `$rangeLt` | `<<` |
| `$rangeLte` | `&<` |
| `$rangeAdjacent` | `-|-` |

### Full-Text Search
```json
{
  "description": {
    "$textSearch": {
      "query": "phone & flagship",
      "type": "websearch",
      "config": "english"
    }
  }
}
```
Types: `plain`, `phrase`, `websearch`, or omit for basic.

### Logical
```json
{
  "$or": [
    { "category": { "$eq": "phones" } },
    { "featured": { "$eq": true } }
  ],
  "$not": {
    "discontinued": { "$eq": true }
  }
}
```

### Match (shorthand for multiple $eq)
```json
{
  "$match": {
    "category": "electronics",
    "status": "active"
  }
}
```

### Raw Filter (escape hatch)
```json
{
  "$filter": {
    "column": "custom_col",
    "operator": "custom_op",
    "value": "val"
  }
}
```

---

## Select Formats

```json
"select": [
  "id",
  "name",
  { "full_name": { "as": "name" } },
  { "metadata": { "path": "$.theme.color", "as": "color" } },
  { "tags": { "path": "$[0]" } }
]
```

JSONPath for JSON/array access:
- `$.key` - object property
- `$.nested.key` - nested property
- `$[0]` - array index
- `$.items[0].name` - combined

---

## Order Format

Always array:

```json
"order": [
  { "column": "price", "direction": "asc" },
  { "column": "created_at", "direction": "desc", "nullsFirst": true }
]
```

| Field | Type | Default |
|-------|------|---------|
| `column` | string | required |
| `direction` | `"asc"` \| `"desc"` | `"asc"` |
| `nullsFirst` | boolean | `false` |

---

## Embedding (Resource Joins)

Embed related tables via foreign key relationships. Maps to PostgREST resource embedding / `table(cols)` syntax.

### Structure

```json
"embed": {
  "table_name": {
    "as": "alias",
    "join": "left",
    "hint": "fk_column",
    "spread": false,
    "select": [...],
    "embed": {...},
    "where": {...},
    "order": [...],
    "limit": 10,
    "offset": 0
  }
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `as` | string | table name | Alias in response |
| `join` | `"left"` \| `"inner"` | `"left"` | Join type — `inner` excludes parent rows without matches |
| `hint` | string | auto-detect | FK column name for disambiguation |
| `spread` | boolean | `false` | Flatten embedded fields into parent |
| `select` | array | `["*"]` | Columns from related table |
| `embed` | object | — | Nested embedding (recursive) |
| `where` | object | — | Filter on embedded resource |
| `order` | array | — | Order within embedded resource |
| `limit` | number | — | Limit embedded rows |
| `offset` | number | — | Offset embedded rows |

### Join Types

**Left (default)** — all parent rows; embedded is `null`/`[]` if no match.

```json
{ "embed": { "actors": { "select": ["name"] } } }
```

**Inner** — only parent rows with matching embedded records.

```json
{ "embed": { "actors": { "join": "inner", "select": ["name"] } } }
```

→ `select('title, actors!inner(name)')` in supabase-js

### Disambiguation

When multiple FKs exist between two tables, use `hint` to specify which.

```json
{
  "from": "orders",
  "select": ["name"],
  "embed": {
    "addresses": [
      { "as": "billing_address", "hint": "billing_address_id", "select": ["name"] },
      { "as": "shipping_address", "hint": "shipping_address_id", "select": ["name"] }
    ]
  }
}
```

→ `select('name, billing_address:addresses!billing_address_id(name), shipping_address:addresses!shipping_address_id(name)')` in supabase-js

### Spread

Flatten embedded fields into parent object instead of nesting.

```json
{
  "from": "films",
  "select": ["title"],
  "embed": {
    "directors": {
      "spread": true,
      "select": [{ "first_name": { "as": "director_name" } }]
    }
  }
}
```

Result: `{ "title": "...", "director_name": "John" }` instead of `{ "title": "...", "directors": { ... } }`

→ `select('title, ...directors(director_name:first_name)')` in supabase-js

### Nested Embedding

Embed within embeds recursively.

```json
{
  "from": "actors",
  "select": ["first_name"],
  "embed": {
    "roles": {
      "select": ["character"],
      "embed": {
        "films": { "select": ["title", "year"] }
      }
    }
  }
}
```

→ `select('first_name, roles(character, films(title, year))')` in supabase-js

### Embedded Filters vs Top-Level Filters

**Embedded filter** — shapes related data, does NOT filter parent:

```json
{
  "from": "films",
  "select": ["*"],
  "embed": {
    "actors": {
      "select": ["*"],
      "where": { "first_name": { "$eq": "Jehanne" } }
    }
  }
}
```

**Top-level filter via `!inner`** — filters parent rows:

```json
{
  "from": "films",
  "select": ["title"],
  "embed": {
    "actors": {
      "join": "inner",
      "select": ["*"],
      "where": { "first_name": { "$eq": "Jehanne" } }
    }
  }
}
```

---

## $meta Fields

| Field | Type | Operations | Description |
|-------|------|------------|-------------|
| `cardinality` | string | query/rpc | `"one"` (error if ≠1), `"maybe"` (null if 0, error if >1), `"many"` (default) |
| `count` | string | all | `"exact"`, `"planned"`, `"estimated"` |
| `head` | boolean | query | Headers only, no body |
| `maxAffected` | number | update/delete | Limit affected rows |
| `rollback` | boolean | all | Return data but rollback transaction |
| `defaultsToNull` | boolean | insert/upsert | Insert null for missing columns |
| `explain` | object | query | Query plan options |
| `headers` | object | all | Custom HTTP headers (for transport) |

### Explain options
```json
{
  "$meta": {
    "explain": {
      "analyze": true,
      "verbose": false,
      "settings": true,
      "buffers": true,
      "wal": false
    }
  }
}
```

---

## Root Fields Summary

| Field | Type | Operations | Description |
|-------|------|------------|-------------|
| `type` | string | all | `query`/`insert`/`update`/`delete`/`upsert`/`rpc` |
| `from` | string | query/insert/update/delete/upsert | Table or view |
| `function` | string | rpc | Function name |
| `schema` | string | all | Schema name |
| `select` | array | all | Columns to return |
| `embed` | object | query/insert/update/delete | Embedded relations (resource joins) |
| `where` | object | query/update/delete/rpc | Filters |
| `values` | object/array | insert/update/upsert | Data |
| `args` | object | rpc | Function arguments |
| `order` | array | query/rpc | Sort |
| `limit` | number | query/rpc | Row limit |
| `offset` | number | query/rpc | Row offset |
| `onConflict` | string | upsert | Conflict column |
| `ignoreDuplicates` | boolean | upsert | Skip vs update |
| `$meta` | object | all | Metadata |

---

## Examples

### Simple query
```json
{
  "from": "users",
  "select": ["id", "email"],
  "where": { "status": { "$eq": "active" } },
  "limit": 10
}
```

### Query with embedding
```json
{
  "from": "posts",
  "select": ["id", "title"],
  "embed": {
    "author": { "select": ["name"] }
  },
  "where": {
    "published": { "$eq": true },
    "$or": [
      { "featured": { "$eq": true } },
      { "views": { "$gt": 1000 } }
    ]
  }
}
```

### Get single row
```json
{
  "from": "users",
  "select": ["id", "email", "profile"],
  "where": { "id": { "$eq": 123 } },
  "$meta": { "cardinality": "one" }
}
```

### Complex filters
```json
{
  "from": "products",
  "select": ["id", "name", "price"],
  "where": {
    "name": { "$ilike": "%phone%" },
    "price": { "$gte": 100, "$lte": 500 },
    "tags": { "$contains": ["featured"] },
    "deleted_at": { "$eq": null },
    "$or": [
      { "category": { "$eq": "electronics" } },
      { "featured": { "$eq": true } }
    ]
  },
  "order": [{ "column": "price", "direction": "asc" }],
  "limit": 20
}
```

### JSON field access
```json
{
  "from": "users",
  "select": [
    "id",
    { "settings": { "path": "$.theme", "as": "theme" } },
    { "settings": { "path": "$.notifications.email", "as": "email_notif" } }
  ],
  "where": {
    "settings": {
      "path": "$.theme",
      "$eq": "dark"
    }
  }
}
```

### Bulk insert with return
```json
{
  "type": "insert",
  "from": "tasks",
  "values": [
    { "title": "Task 1", "status": "pending" },
    { "title": "Task 2", "status": "pending" }
  ],
  "select": ["id", "title", "created_at"]
}
```

### Safe update with limit
```json
{
  "type": "update",
  "from": "orders",
  "values": { "status": "shipped" },
  "where": {
    "status": { "$eq": "pending" },
    "paid": { "$eq": true }
  },
  "select": ["id", "status"],
  "$meta": { "maxAffected": 100 }
}
```

---

## Parser Notes

### NULL handling
- `$eq: null` → `col IS NULL`
- `$neq: null` → `col IS NOT NULL`
- All other values use standard `=` / `<>`

### JSONPath in where
When filtering on JSON fields:
```json
{
  "metadata": {
    "path": "$.status",
    "$eq": "active"
  }
}
```
→ `metadata->>'status' = 'active'` (PostgreSQL)
→ `json_extract(metadata, '$.status') = 'active'` (SQLite)

### Security
- Parameterize all values
- Whitelist column/table names
- Enforce `maxAffected` limits
- Validate JSONPath syntax
- Sanitize `$filter` operators

---

## Coverage

All 66 methods mapped:

- **Core (6+):** from, select, insert, update, delete, upsert, embed (joins)
- **Comparison (10):** eq, neq, gt, gte, lt, lte, in, notIn, is→eq, isDistinct
- **Pattern (6):** like, ilike, likeAllOf, likeAnyOf, ilikeAllOf, ilikeAnyOf
- **Regex (2):** regex, iregex
- **Array/JSON (3):** contains, containedBy, overlaps
- **Range (5):** rangeGt, rangeGte, rangeLt, rangeLte, rangeAdjacent
- **Advanced (5):** textSearch, match, or, not, filter
- **Transforms (5):** order, limit, offset (range merged), single/maybeSingle→cardinality
- **Response (3):** csv/geojson/explain (csv/geojson via transport)
- **Utilities (3):** abortSignal (runtime), setHeader→headers, throwOnError (runtime)
- **Specialized:** rollback, maxAffected, schema, rpc, etc.

---

## Open Questions

1. Should `path` in where use separate field or inline like `"metadata.$.theme": { "$eq": "dark" }`?
2. `$match` redundant? Just shorthand for multiple `$eq`
3. `$filter` needed? Security risk vs flexibility tradeoff
4. Nested `$or`/`$and` - current spec allows `$or` at top level, nested `$or` in `$or`?
