# Response Shaping

**Category:** MEDIUM Priority | **Status:** ✅ Complete
**Methods:** 3 (csv, geojson, explain)

Methods for changing the response format of query results.

---

## `csv()`

**Priority:** Medium | **SQLite:** ✅ Full

**Signature:**
```ts
csv(): PostgrestBuilder<string>
```

**Purpose:** Return query results as CSV text instead of JSON.

**Parameters:** None

**Returns:** Builder with result type changed to `string` (CSV format).

**Usage:**
```ts
// Get results as CSV
const { data, error } = await client
  .from('users')
  .select('id,name,email')
  .csv()
// data: "id,name,email\n1,John,john@example.com\n2,Jane,jane@example.com"

// With filters
const { data } = await client
  .from('products')
  .select('*')
  .eq('category', 'electronics')
  .csv()

// Download as file
const { data } = await client
  .from('reports')
  .select('*')
  .order('date')
  .csv()

if (data) {
  const blob = new Blob([data], { type: 'text/csv' })
  const url = URL.createObjectURL(blob)
  // Trigger download...
}
```

**PostgREST Mapping:**
- Header: `Accept: text/csv`
- Response: Plain text CSV with headers
- Format:
  ```
  column1,column2,column3
  value1,value2,value3
  value1,value2,value3
  ```

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full (requires formatting)
- **SQL:** Standard query, format output
- **Notes:**
  - No native CSV output in SQLite
  - Execute query, format rows as CSV manually
  - Handle escaping: quotes, commas, newlines
- **Implementation:**
  ```ts
  const rows = executeQuery('SELECT id, name, email FROM users')

  // Build CSV
  const headers = Object.keys(rows[0]).join(',')
  const lines = rows.map(row =>
    Object.values(row)
      .map(val => {
        // Escape quotes and wrap if contains comma/quote/newline
        if (val == null) return ''
        const str = String(val)
        if (str.includes(',') || str.includes('"') || str.includes('\n')) {
          return `"${str.replace(/"/g, '""')}"`
        }
        return str
      })
      .join(',')
  )
  const csv = [headers, ...lines].join('\n')
  ```

**Edge Cases:**
- Empty result: Just headers (no data rows)
- NULL values: Typically empty string in CSV
- Commas in data: Wrapped in quotes, escaped
- Quotes in data: Doubled: `"value with ""quotes"""`
- Newlines in data: Wrapped in quotes
- Array/JSON columns: Serialized to string

**Related:** geojson(), select()

**Source:** PostgrestTransformBuilder.ts:240-245

---

## `geojson()`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
geojson(): PostgrestBuilder<Record<string, unknown>>
```

**Purpose:** Return query results in GeoJSON format for geographic data.

**Parameters:** None

**Returns:** Builder with result type changed to GeoJSON object.

**Usage:**
```ts
// Get geographic data as GeoJSON
const { data, error } = await client
  .from('locations')
  .select('*')
  .geojson()
// data: { type: 'FeatureCollection', features: [...] }

// With spatial filter
const { data } = await client
  .from('cities')
  .select('*')
  .eq('country', 'USA')
  .geojson()

// Use with mapping library
const { data } = await client
  .from('restaurants')
  .select('id,name,location')
  .geojson()

if (data) {
  // Load into Leaflet, Mapbox, etc.
  L.geoJSON(data).addTo(map)
}
```

**PostgREST Mapping:**
- Header: `Accept: application/geo+json`
- Response: GeoJSON FeatureCollection
- Format:
  ```json
  {
    "type": "FeatureCollection",
    "features": [
      {
        "type": "Feature",
        "geometry": { "type": "Point", "coordinates": [-73.935242, 40.730610] },
        "properties": { "id": 1, "name": "Location Name" }
      }
    ]
  }
  ```

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial (requires manual formatting)
- **SQL:** Query geometry data, format as GeoJSON
- **Notes:**
  - SQLite has no native GeoJSON support
  - Requires SpatiaLite extension for geometry types
  - Manual conversion from geometry to GeoJSON
- **Implementation:**
  ```ts
  const rows = executeQuery('SELECT id, name, AsGeoJSON(geom) as geometry FROM locations')

  const geojson = {
    type: 'FeatureCollection',
    features: rows.map(row => ({
      type: 'Feature',
      geometry: JSON.parse(row.geometry),
      properties: {
        id: row.id,
        name: row.name,
        // other non-geometry columns
      }
    }))
  }
  ```
- **Workarounds:**
  - Use SpatiaLite extension for geometry types
  - Use AsGeoJSON() function to convert geometry
  - Or store geometry as GeoJSON text directly
  - Build FeatureCollection wrapper manually

**Edge Cases:**
- No geometry column: May not work or return plain JSON
- Multiple geometry columns: Typically uses first/primary
- Non-geometry columns: Become feature properties
- Empty result: `{ type: 'FeatureCollection', features: [] }`

**Related:** csv(), select()

**Source:** PostgrestTransformBuilder.ts:247-253

---

## `explain(options)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
explain(options?: {
  analyze?: boolean
  verbose?: boolean
  settings?: boolean
  buffers?: boolean
  wal?: boolean
  format?: 'json' | 'text'
}): PostgrestBuilder<Record<string, unknown>[] | string>
```

**Purpose:** Return query execution plan instead of results. For performance debugging.

**Parameters:**
- `options.analyze` (boolean, optional) - Execute query and include actual runtime stats. Default: `false`
- `options.verbose` (boolean, optional) - Include detailed output. Default: `false`
- `options.settings` (boolean, optional) - Include config parameters. Default: `false`
- `options.buffers` (boolean, optional) - Include buffer usage stats. Default: `false`
- `options.wal` (boolean, optional) - Include WAL record generation info. Default: `false`
- `options.format` (string, optional) - Output format: `'text'` (default) or `'json'`

**Returns:**
- If `format: 'text'`: `string` (plain text plan)
- If `format: 'json'`: `Record<string, unknown>[]` (structured plan)

**Usage:**
```ts
// Basic explain
const { data } = await client
  .from('users')
  .select('*')
  .eq('status', 'active')
  .explain()
// data: "Seq Scan on users  (cost=0.00..35.50 rows=10 width=100)\n  Filter: (status = 'active')"

// Analyze (execute + stats)
const { data } = await client
  .from('products')
  .select('*')
  .gte('price', 100)
  .explain({ analyze: true })
// Includes actual time, actual rows, etc.

// JSON format
const { data } = await client
  .from('orders')
  .select('*')
  .order('created_at')
  .limit(100)
  .explain({ format: 'json', analyze: true })
// data: [{ "Plan": { "Node Type": "Limit", ... } }]

// All options
const { data } = await client
  .from('large_table')
  .select('*')
  .explain({
    analyze: true,
    verbose: true,
    settings: true,
    buffers: true,
    format: 'json'
  })
```

**PostgREST Mapping:**
- Header: `Accept: application/vnd.pgrst.plan+text` or `application/vnd.pgrst.plan+json`
- Header format: `Accept: application/vnd.pgrst.plan+<format>; for="<original-accept>"; options=<flags>;`
- Requires `db_plan_enabled` setting in PostgREST config
- Postgres EXPLAIN executed server-side

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial (different format)
- **SQL:** `EXPLAIN QUERY PLAN SELECT ...`
- **Notes:**
  - SQLite has `EXPLAIN QUERY PLAN` but different output format
  - Much simpler than Postgres EXPLAIN
  - No equivalent for analyze, buffers, wal options
  - Output format differs significantly
- **Implementation:**
  ```ts
  const plan = executeQuery('EXPLAIN QUERY PLAN SELECT * FROM users WHERE status = ?', ['active'])

  // SQLite output:
  // [
  //   { selectid: 0, order: 0, from: 0, detail: 'SCAN TABLE users' }
  // ]

  // Format as text
  const text = plan.map(row => row.detail).join('\n')
  ```
- **Workarounds:**
  - Map SQLite query plan to similar text format
  - `analyze` option: Not available (SQLite doesn't execute in EXPLAIN)
  - `verbose`, `settings`, `buffers`, `wal`: Not applicable
  - `format: 'json'`: Return SQLite plan rows as-is

**Edge Cases:**
- Requires `db_plan_enabled` config in PostgREST
- Query is NOT executed unless `analyze: true`
- Performance: `analyze` executes query (may be slow)
- Format: JSON plan structure is complex, nested

**Related:** select()

**Source:** PostgrestTransformBuilder.ts:255-315

---

## Summary

Response shaping methods with SQLite compatibility:

| Method | PostgREST | SQLite | Notes |
|--------|-----------|--------|-------|
| `csv()` | `Accept: text/csv` | Manual CSV formatting | ✅ Full support with formatting |
| `geojson()` | `Accept: application/geo+json` | SpatiaLite + manual | ⚠️ Requires SpatiaLite extension |
| `explain()` | `EXPLAIN` header | `EXPLAIN QUERY PLAN` | ⚠️ Different format, fewer options |

**Implementation Notes:**
- `csv()`: Straightforward formatting logic
- `geojson()`: Needs geometry extension + GeoJSON wrapper
- `explain()`: Different output format, limited options
