# Database API Overview

**Category:** Reference | **Status:** ✅ Complete

Complete reference of all ~66 Database (postgrest-js) API methods.

---

## Quick Reference by Category

### Core Operations (6 methods)
**Start queries and perform CRUD**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `from()` | Start query on table/view | HIGH | [core-operations.md](./core-operations.md#fromrelation) |
| `select()` | Fetch rows | HIGH | [core-operations.md](./core-operations.md#selectcolumns-options) |
| `insert()` | Create rows | HIGH | [core-operations.md](./core-operations.md#insertvalues-options) |
| `update()` | Modify rows | HIGH | [core-operations.md](./core-operations.md#updatevalues-options) |
| `delete()` | Remove rows | HIGH | [core-operations.md](./core-operations.md#deleteoptions) |
| `upsert()` | Insert or update | HIGH | [core-operations.md](./core-operations.md#upsertvalues-options) |

### Comparison Filters (10 methods)
**Basic comparison operators**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `eq()` | Equals | HIGH | [filters-comparison.md](./filters-comparison.md#eqcolumn-value) |
| `neq()` | Not equals | HIGH | [filters-comparison.md](./filters-comparison.md#neqcolumn-value) |
| `gt()` | Greater than | HIGH | [filters-comparison.md](./filters-comparison.md#gtcolumn-value) |
| `gte()` | Greater than or equal | HIGH | [filters-comparison.md](./filters-comparison.md#gtecolumn-value) |
| `lt()` | Less than | HIGH | [filters-comparison.md](./filters-comparison.md#ltcolumn-value) |
| `lte()` | Less than or equal | HIGH | [filters-comparison.md](./filters-comparison.md#ltecolumn-value) |
| `in()` | In array | HIGH | [filters-comparison.md](./filters-comparison.md#incolumn-values) |
| `notIn()` | Not in array | HIGH | [filters-comparison.md](./filters-comparison.md#notincolumn-values) |
| `is()` | IS NULL / boolean | HIGH | [filters-comparison.md](./filters-comparison.md#iscolumn-value) |
| `isDistinct()` | IS DISTINCT FROM | HIGH | [filters-comparison.md](./filters-comparison.md#isdistinctcolumn-value) |

### Transforms (5 methods)
**Sort, paginate, shape results**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `order()` | Sort results | HIGH | [transforms.md](./transforms.md#ordercolumn-options) |
| `limit()` | Limit rows | HIGH | [transforms.md](./transforms.md#limitcount-options) |
| `range()` | Paginate | HIGH | [transforms.md](./transforms.md#rangefrom-to-options) |
| `single()` | Get one object | HIGH | [transforms.md](./transforms.md#single) |
| `maybeSingle()` | Get one or null | HIGH | [transforms.md](./transforms.md#maybesingle) |

### Pattern Matching Filters (6 methods)
**Text pattern search**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `like()` | LIKE (case-sensitive) | MEDIUM | [filters-pattern-matching.md](./filters-pattern-matching.md#likecolumn-pattern) |
| `ilike()` | LIKE (case-insensitive) | MEDIUM | [filters-pattern-matching.md](./filters-pattern-matching.md#ilikecolumn-pattern) |
| `likeAllOf()` | Match all patterns | MEDIUM | [filters-pattern-matching.md](./filters-pattern-matching.md#likeallofcolumn-patterns) |
| `likeAnyOf()` | Match any pattern | MEDIUM | [filters-pattern-matching.md](./filters-pattern-matching.md#likeanyofcolumn-patterns) |
| `ilikeAllOf()` | Match all (case-insensitive) | MEDIUM | [filters-pattern-matching.md](./filters-pattern-matching.md#ilikeallofcolumn-patterns) |
| `ilikeAnyOf()` | Match any (case-insensitive) | MEDIUM | [filters-pattern-matching.md](./filters-pattern-matching.md#ilikeanyofcolumn-patterns) |

### Array & JSON Filters (3 methods)
**Array, JSON, range operations**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `contains()` | Column contains value | MEDIUM | [filters-array-json.md](./filters-array-json.md#containscolumn-value) |
| `containedBy()` | Column contained by value | MEDIUM | [filters-array-json.md](./filters-array-json.md#containedbycolumn-value) |
| `overlaps()` | Column overlaps value | MEDIUM | [filters-array-json.md](./filters-array-json.md#overlapscolumn-value) |

### Advanced Filters (5 methods)
**Full-text search, logical operators**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `textSearch()` | Full-text search | MEDIUM | [filters-advanced.md](./filters-advanced.md#textsearchcolumn-query-options) |
| `match()` | Multiple equality | MEDIUM | [filters-advanced.md](./filters-advanced.md#matchquery) |
| `or()` | Logical OR | MEDIUM | [filters-advanced.md](./filters-advanced.md#orfilters-options) |
| `not()` | Negate filter | MEDIUM | [filters-advanced.md](./filters-advanced.md#notcolumn-operator-value) |
| `filter()` | Custom filter | MEDIUM | [filters-advanced.md](./filters-advanced.md#filtercolumn-operator-value) |

### Response Shaping (3 methods)
**Change output format**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `csv()` | CSV output | MEDIUM | [response-shaping.md](./response-shaping.md#csv) |
| `geojson()` | GeoJSON output | MEDIUM | [response-shaping.md](./response-shaping.md#geojson) |
| `explain()` | Query plan | MEDIUM | [response-shaping.md](./response-shaping.md#explainoptions) |

### Utilities (3 methods)
**Request control**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `abortSignal()` | Cancel request | MEDIUM | [utilities.md](./utilities.md#abortsignalsignal) |
| `setHeader()` | Custom headers | MEDIUM | [utilities.md](./utilities.md#setheadername-value) |
| `throwOnError()` | Exception-based errors | MEDIUM | [utilities.md](./utilities.md#throwonerror) |

### Range Filters (5 methods)
**PostgreSQL range types**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `rangeGt()` | Range greater than | LOW | [filters-range.md](./filters-range.md#rangegtcolumn-range) |
| `rangeGte()` | Range greater/equal | LOW | [filters-range.md](./filters-range.md#rangegtecolumn-range) |
| `rangeLt()` | Range less than | LOW | [filters-range.md](./filters-range.md#rangeltcolumn-range) |
| `rangeLte()` | Range less/equal | LOW | [filters-range.md](./filters-range.md#rangeltecolumn-range) |
| `rangeAdjacent()` | Range adjacent | LOW | [filters-range.md](./filters-range.md#rangeadjacentcolumn-range) |

### Specialized Methods (8+ methods)
**Regex, control, type utilities**

| Method | Purpose | Priority | File |
|--------|---------|----------|------|
| `regexMatch()` | Regex (case-sensitive) | LOW | [filters-specialized.md](./filters-specialized.md#regexmatchcolumn-pattern) |
| `regexIMatch()` | Regex (case-insensitive) | LOW | [filters-specialized.md](./filters-specialized.md#regeximatchcolumn-pattern) |
| `rollback()` | Rollback transaction | LOW | [filters-specialized.md](./filters-specialized.md#rollback) |
| `maxAffected()` | Max affected rows | LOW | [filters-specialized.md](./filters-specialized.md#maxaffectedvalue) |
| `returns()` | Override types (deprecated) | LOW | [filters-specialized.md](./filters-specialized.md#returnsnewresult) |
| `overrideTypes()` | Override types | LOW | [filters-specialized.md](./filters-specialized.md#overridetypesnewresult-options) |
| `schema()` | Switch schema | LOW | [filters-specialized.md](./filters-specialized.md#schemaschemaname) |
| `rpc()` | Call stored procedure | MED/LOW | [filters-specialized.md](./filters-specialized.md#rpcfn-args-options) |

---

## Common Patterns

### Basic CRUD

```ts
// Read
const { data } = await client.from('users').select('*')

// Read with filter
const { data } = await client.from('users').select('*').eq('status', 'active')

// Create
const { data, error } = await client.from('users').insert({ name: 'John' }).select()

// Update
const { data } = await client.from('users').update({ name: 'Jane' }).eq('id', 1).select()

// Delete
const { error } = await client.from('users').delete().eq('id', 1)

// Upsert
const { data } = await client.from('users').upsert({ id: 1, name: 'John' }).select()
```

### Filtering & Sorting

```ts
// Multiple filters (AND)
const { data } = await client
  .from('posts')
  .select('*')
  .eq('status', 'published')
  .gte('views', 100)
  .order('created_at', { ascending: false })
  .limit(10)

// OR filter
const { data } = await client
  .from('users')
  .select('*')
  .or('role.eq.admin,role.eq.moderator')

// Pattern matching
const { data } = await client
  .from('products')
  .select('*')
  .ilike('name', '%phone%')

// Range / Pagination
const { data } = await client
  .from('posts')
  .select('*')
  .order('created_at')
  .range(0, 9) // First 10 items
```

### Relationships

```ts
// Select related data
const { data } = await client
  .from('users')
  .select('*, posts(*), comments(*)')

// Filter on related table
const { data } = await client
  .from('users')
  .select('*, posts!inner(*)')
  .eq('posts.status', 'published')
```

### Advanced Usage

```ts
// Count only
const { count } = await client
  .from('users')
  .select('*', { count: 'exact', head: true })

// Single result
const { data } = await client
  .from('users')
  .select('*')
  .eq('id', 1)
  .single()

// CSV export
const { data } = await client
  .from('reports')
  .select('*')
  .csv()

// With error handling
const { data } = await client
  .from('users')
  .select('*')
  .throwOnError()

// Cancellable request
const controller = new AbortController()
const { data } = await client
  .from('users')
  .select('*')
  .abortSignal(controller.signal)
```

---

## Method Chaining

Methods chain in logical order:

```
client
  .from(table)                // Start: choose table
  .select() / .insert() / ... // Operation: CRUD
  .eq() / .gt() / ...         // Filters: WHERE conditions
  .order() / .limit()         // Transforms: ORDER BY, LIMIT
  .single() / .csv()          // Format: response shape
  .throwOnError()             // Control: error handling
```

Some methods return different builder types:
- `from()` → `PostgrestQueryBuilder`
- `select() / insert() / ...` → `PostgrestFilterBuilder` (can add filters)
- `single() / csv() / ...` → `PostgrestBuilder` (final, execute)

---

## See Also

- [SQLite Compatibility Matrix](./sqlite-compatibility-matrix.md) - Full compatibility reference
- [Core Operations](./core-operations.md) - CRUD methods (HIGH priority)
- [Comparison Filters](./filters-comparison.md) - Basic filters (HIGH priority)
- [Transforms](./transforms.md) - Sorting and pagination (HIGH priority)
