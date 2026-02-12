# Database API Object Representation

Unified JSON object format covering all 66 Database API methods for building query parsers/executors.

---

## Core Concept

Single object structure with `type` field determining operation mode:
- `query` (default) - SELECT operations
- `insert` - INSERT operations
- `update` - UPDATE operations
- `delete` - DELETE operations
- `upsert` - INSERT ... ON CONFLICT operations
- `rpc` - Function calls

---

## Complete Specification

### 1. Query Operations (SELECT)

```json
{
  "type": "query",
  "from": "products",
  "schema": "public",

  "select": [
    "id",
    "name",
    "price",
    {
      "description": {
        "as": "desc"
      }
    },
    {
      "metadata": {
        "type": "json",
        "operator": "->",
        "path": "theme",
        "as": "theme_value"
      }
    },
    {
      "tags": {
        "type": "array",
        "operator": "->",
        "index": 0
      }
    }
  ],

  "with": {
    "categories": {
      "select": ["id", "name"],
      "where": {
        "active": { "$is": true }
      },
      "limit": 5
    },
    "reviews": {
      "as": "product_reviews",
      "select": ["rating", "comment", "created_at"],
      "order": { "created_at": "desc" }
    }
  },

  "where": {
    "name": {
      "$like": "%iPhone%",
      "$ilike": "%android%",
      "$in": ["iPhone 15", "Galaxy S24"],
      "$notIn": ["iPhone 14"],
      "$eq": "iPhone 15 Pro",
      "$neq": "iPhone 14",
      "$gt": "iPhone",
      "$gte": "iPhone 15",
      "$lt": "Samsung",
      "$lte": "Samsung Z",
      "$is": null,
      "$isDistinct": null,
      "$likeAllOf": ["%phone%", "%pro%"],
      "$likeAnyOf": ["%phone%", "%tablet%"],
      "$ilikeAllOf": ["%iphone%", "%pro%"],
      "$ilikeAnyOf": ["%iphone%", "%android%"],
      "$regexMatch": "^iPhone.*Pro$",
      "$regexIMatch": "^iphone.*pro$"
    },

    "price": {
      "$gt": 100,
      "$gte": 100,
      "$lt": 2000,
      "$lte": 2000
    },

    "tags": {
      "$contains": ["featured", "new"],
      "$containedBy": ["electronics", "mobile", "featured"],
      "$overlaps": ["featured", "sale"]
    },

    "metadata": {
      "$contains": { "color": "black" },
      "$containedBy": { "color": "black", "storage": "256GB" }
    },

    "description": {
      "$textSearch": {
        "query": "flagship & premium",
        "type": "plain",
        "config": "english"
      }
    },

    "stock_range": {
      "$rangeGt": "[10,100)",
      "$rangeGte": "[10,100)",
      "$rangeLt": "[10,100)",
      "$rangeLte": "[10,100)",
      "$rangeAdjacent": "[10,100)"
    },

    "$or": [
      { "category": { "$eq": "phones" } },
      { "featured": { "$is": true } }
    ],

    "$not": {
      "discontinued": { "$is": true }
    },

    "$match": {
      "category": "electronics",
      "status": "active"
    },

    "$filter": {
      "custom_column": {
        "operator": "custom_op",
        "value": "custom_value"
      }
    }
  },

  "order": [
    { "price": "asc" },
    { "name": "desc" },
    { "created_at": { "order": "desc", "nullsFirst": true } }
  ],

  "limit": 50,
  "offset": 0,

  "range": {
    "from": 10,
    "to": 59
  },

  "$meta": {
    "single": false,
    "maybeSingle": false,
    "count": "exact",
    "format": "json",
    "head": false,
    "maxAffected": null,
    "rollback": false,
    "headers": {
      "Prefer": "return=representation",
      "Accept": "application/json"
    },
    "abortSignal": null,
    "throwOnError": false,
    "explain": null
  }
}
```

### 2. Insert Operations

```json
{
  "type": "insert",
  "from": "products",
  "schema": "public",

  "values": [
    {
      "name": "iPhone 15",
      "price": 999,
      "category": "phones"
    },
    {
      "name": "Galaxy S24",
      "price": 899,
      "category": "phones"
    }
  ],

  "select": ["id", "name", "created_at"],

  "$meta": {
    "defaultsToNull": true,
    "count": "exact",
    "headers": {
      "Prefer": "return=representation,resolution=merge-duplicates"
    }
  }
}
```

### 3. Update Operations

```json
{
  "type": "update",
  "from": "products",
  "schema": "public",

  "values": {
    "price": 899,
    "updated_at": "2026-02-02T00:00:00Z",
    "tags": ["sale", "featured"]
  },

  "where": {
    "id": { "$eq": 123 },
    "status": { "$eq": "active" }
  },

  "select": ["id", "name", "price", "updated_at"],

  "$meta": {
    "count": "exact",
    "maxAffected": 1,
    "headers": {
      "Prefer": "return=representation"
    }
  }
}
```

### 4. Delete Operations

```json
{
  "type": "delete",
  "from": "products",
  "schema": "public",

  "where": {
    "id": { "$in": [1, 2, 3] },
    "deleted_at": { "$is": null }
  },

  "select": ["id", "name"],

  "$meta": {
    "count": "exact",
    "maxAffected": 10,
    "headers": {
      "Prefer": "return=representation"
    }
  }
}
```

### 5. Upsert Operations

```json
{
  "type": "upsert",
  "from": "products",
  "schema": "public",

  "values": [
    {
      "id": 1,
      "name": "iPhone 15",
      "price": 999
    },
    {
      "id": 2,
      "name": "Galaxy S24",
      "price": 899
    }
  ],

  "onConflict": "id",
  "ignoreDuplicates": false,

  "select": ["id", "name", "price", "updated_at"],

  "$meta": {
    "defaultsToNull": false,
    "count": "exact",
    "headers": {
      "Prefer": "return=representation,resolution=merge-duplicates"
    }
  }
}
```

### 6. RPC (Function Calls)

```json
{
  "type": "rpc",
  "function": "calculate_discount",
  "schema": "public",

  "args": {
    "product_id": 123,
    "discount_percent": 15,
    "apply_tax": true
  },

  "select": ["final_price", "tax_amount"],

  "where": {
    "result_status": { "$eq": "success" }
  },

  "order": [
    { "final_price": "asc" }
  ],

  "limit": 10,

  "$meta": {
    "count": null,
    "single": false,
    "headers": {
      "Prefer": "params=single-object"
    }
  }
}
```

### 7. Response Format Variants

```json
{
  "type": "query",
  "from": "locations",

  "select": ["id", "name", "coordinates"],

  "$meta": {
    "format": "geojson",
    "headers": {
      "Accept": "application/geo+json"
    }
  }
}
```

```json
{
  "type": "query",
  "from": "products",

  "select": ["id", "name", "price"],

  "$meta": {
    "format": "csv",
    "headers": {
      "Accept": "text/csv"
    }
  }
}
```

```json
{
  "type": "query",
  "from": "products",

  "select": ["id", "name"],

  "where": {
    "price": { "$gt": 100 }
  },

  "$meta": {
    "explain": {
      "analyze": true,
      "verbose": false,
      "settings": true,
      "buffers": true,
      "wal": false,
      "format": "json"
    },
    "headers": {
      "Accept": "application/vnd.pgrst.plan+json"
    }
  }
}
```

---

## Operator Reference

### Comparison Operators
- `$eq` - equals (eq)
- `$neq` - not equals (neq)
- `$gt` - greater than (gt)
- `$gte` - greater than or equal (gte)
- `$lt` - less than (lt)
- `$lte` - less than or equal (lte)
- `$in` - in array (in)
- `$notIn` - not in array (not.in)
- `$is` - IS (for NULL checks)
- `$isDistinct` - IS DISTINCT FROM (isdistinct)

### Pattern Matching
- `$like` - LIKE pattern (like)
- `$ilike` - ILIKE pattern (ilike)
- `$likeAllOf` - matches all patterns (like(all))
- `$likeAnyOf` - matches any pattern (like(any))
- `$ilikeAllOf` - matches all patterns case-insensitive (ilike(all))
- `$ilikeAnyOf` - matches any pattern case-insensitive (ilike(any))

### Regex
- `$regexMatch` - regex match case-sensitive (~)
- `$regexIMatch` - regex match case-insensitive (~*)

### Array/JSON/Range
- `$contains` - contains all elements (cs)
- `$containedBy` - contained by (cd)
- `$overlaps` - has common elements (ov)
- `$rangeGt` - range strictly right (sr)
- `$rangeGte` - range not extends left (nxl)
- `$rangeLt` - range strictly left (sl)
- `$rangeLte` - range not extends right (nxr)
- `$rangeAdjacent` - ranges adjacent (adj)

### Advanced
- `$textSearch` - full-text search (fts/plfts/phfts/wfts)
- `$match` - multiple column equality
- `$or` - logical OR
- `$not` - negate condition
- `$filter` - raw operator passthrough

---

## Field Specifications

### Root Level Fields

| Field | Type | Operations | Description |
|-------|------|------------|-------------|
| `type` | string | all | Operation type: query/insert/update/delete/upsert/rpc |
| `from` | string | query/insert/update/delete/upsert | Table or view name |
| `function` | string | rpc | Function name |
| `schema` | string | all | Schema name (default: public) |
| `select` | array | all | Columns to return |
| `with` | object | query | Embedded resources (relationships) |
| `where` | object | query/update/delete/rpc | Filter conditions |
| `values` | object/array | insert/update/upsert | Data to insert/update |
| `args` | object | rpc | Function arguments |
| `order` | array | query/rpc | Sort specification |
| `limit` | number | query/rpc | Row limit |
| `offset` | number | query/rpc | Row offset |
| `range` | object | query/rpc | Pagination range |
| `onConflict` | string | upsert | Conflict resolution column |
| `ignoreDuplicates` | boolean | upsert | Skip duplicates instead of update |
| `$meta` | object | all | Metadata, headers, response options |

### $meta Fields

| Field | Type | Description |
|-------|------|-------------|
| `single` | boolean | Expect exactly 1 row (error if 0 or >1) |
| `maybeSingle` | boolean | Return single row or null (error if >1) |
| `count` | string | Count algorithm: exact/planned/estimated |
| `format` | string | Response format: json/csv/geojson |
| `head` | boolean | Return headers only (no body) |
| `maxAffected` | number | Max rows allowed for update/delete |
| `rollback` | boolean | Rollback transaction after returning data |
| `defaultsToNull` | boolean | Insert nulls for missing columns |
| `headers` | object | Custom HTTP headers |
| `abortSignal` | AbortSignal | Request cancellation signal |
| `throwOnError` | boolean | Throw errors instead of returning them |
| `explain` | object | Query execution plan options |

### select Array Formats

```json
// Simple column
"select": ["id", "name"]

// Aliased column
"select": [
  {
    "full_name": {
      "as": "name"
    }
  }
]

// JSON operator
"select": [
  {
    "metadata": {
      "type": "json",
      "operator": "->",
      "path": "theme.color",
      "as": "theme_color"
    }
  }
]

// Array operator
"select": [
  {
    "tags": {
      "type": "array",
      "operator": "->",
      "index": 0,
      "as": "first_tag"
    }
  }
]
```

### order Array Formats

```json
// Simple
"order": [
  { "price": "asc" },
  { "name": "desc" }
]

// With options
"order": [
  {
    "created_at": {
      "order": "desc",
      "nullsFirst": true,
      "foreignTable": null
    }
  }
]
```

### $textSearch Format

```json
{
  "$textSearch": {
    "query": "flagship & premium | budget",
    "type": "plain",
    "config": "english"
  }
}
```

Types: `plain` (plfts), `phrase` (phfts), `websearch` (wfts), or null for basic (fts)

### explain Format

```json
{
  "explain": {
    "analyze": true,
    "verbose": false,
    "settings": true,
    "buffers": true,
    "wal": false,
    "format": "json"
  }
}
```

---

## Usage Examples

### Simple Query
```json
{
  "from": "users",
  "select": ["id", "email", "created_at"],
  "where": {
    "status": { "$eq": "active" }
  },
  "order": [{ "created_at": "desc" }],
  "limit": 10
}
```

### Complex Query with Joins
```json
{
  "from": "posts",
  "select": ["id", "title", "content"],
  "with": {
    "author": {
      "select": ["name", "avatar"]
    },
    "comments": {
      "select": ["text", "created_at"],
      "where": {
        "approved": { "$is": true }
      },
      "order": [{ "created_at": "desc" }],
      "limit": 5
    }
  },
  "where": {
    "published": { "$is": true },
    "$or": [
      { "featured": { "$is": true } },
      { "views": { "$gt": 1000 } }
    ]
  },
  "order": [{ "published_at": "desc" }]
}
```

### Bulk Insert
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

### Conditional Update
```json
{
  "type": "update",
  "from": "orders",
  "values": {
    "status": "shipped",
    "shipped_at": "2026-02-02T12:00:00Z"
  },
  "where": {
    "status": { "$eq": "pending" },
    "payment_status": { "$eq": "completed" }
  },
  "select": ["id", "status", "shipped_at"],
  "$meta": {
    "count": "exact",
    "maxAffected": 100
  }
}
```

### Upsert with Conflict Resolution
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

### Safe Delete with Limit
```json
{
  "type": "delete",
  "from": "logs",
  "where": {
    "created_at": { "$lt": "2026-01-01T00:00:00Z" },
    "processed": { "$is": true }
  },
  "select": ["id", "created_at"],
  "$meta": {
    "count": "exact",
    "maxAffected": 1000
  }
}
```

---

## Implementation Notes

### SQLite Translation
- Operators map to SQL: `$eq` → `=`, `$gt` → `>`, `$in` → `IN (...)`, etc.
- `$textSearch` requires FTS5 extension
- `$contains`/`$overlaps` require json1 extension
- `$range*` operators not supported (no range types)
- `$regexMatch` uses REGEXP (requires custom function)

### PostgREST HTTP Mapping
- `where` → query params: `?col=op.value`
- `order` → `?order=col.asc`
- `limit` → `?limit=N`
- `offset` → `?offset=N`
- `range` → `Range` header or `?offset=X&limit=Y`
- `select` → `?select=cols`
- `$meta.count` → `Prefer: count=exact`
- `$meta.single` → `Accept: application/vnd.pgrst.object+json`

### Parser Implementation
1. Validate `type` field (default: "query")
2. Build base SQL from `from`/`function` + `schema`
3. Transform `where` object to SQL WHERE clause recursively
4. Handle `$or`/`$not` as nested conditions
5. Apply `order`/`limit`/`offset`/`range` to SQL
6. For mutations, merge `values` with `where` conditions
7. Apply `select` as RETURNING clause if present
8. Set HTTP headers from `$meta`

### Security Considerations
- Always parameterize values (prevent SQL injection)
- Validate column names against schema (prevent column injection)
- Enforce `maxAffected` limits for updates/deletes
- Sanitize `$filter` operator strings (whitelist only)
- Validate `schema` names (prevent schema traversal)
- Check permissions before executing queries

---

## Coverage Summary

**All 66 methods covered:**

✅ Core Operations (6): from, select, insert, update, delete, upsert
✅ Comparison Filters (10): eq, neq, gt, gte, lt, lte, in, notIn, is, isDistinct
✅ Pattern Matching (6): like, ilike, likeAllOf, likeAnyOf, ilikeAllOf, ilikeAnyOf
✅ Array/JSON (3): contains, containedBy, overlaps
✅ Advanced Filters (5): textSearch, match, or, not, filter
✅ Transforms (5): order, limit, range, single, maybeSingle
✅ Response Shaping (3): csv, geojson, explain
✅ Utilities (3): abortSignal, setHeader, throwOnError
✅ Range Filters (5): rangeGt, rangeGte, rangeLt, rangeLte, rangeAdjacent
✅ Specialized (20): regex, rollback, maxAffected, returns, overrideTypes, schema, rpc, etc.

**Operation Types:** 6 (query, insert, update, delete, upsert, rpc)
**Operators:** 35+ (comparison, pattern, array/JSON, range, advanced)
**Metadata Fields:** 13 (single, count, format, headers, etc.)

---

## Version History

- **1.0.0** (2026-02-02) - Initial specification covering all 66 Database API methods
