# Array & JSON Filters

**Category:** MEDIUM Priority | **Status:** ✅ Complete
**Methods:** 3 (contains, containedBy, overlaps)

Filters for array, JSON, and range column types.

---

## `contains(column, value)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
contains<ColumnName extends string & keyof Row>(
  column: ColumnName,
  value: string | ReadonlyArray<Row[ColumnName]> | Record<string, unknown>
): this
```

**Purpose:** Match rows where column contains all elements in value. For arrays, JSON, and range types.

**Parameters:**
- `column` (string) - Column name (must be array, jsonb, or range type)
- `value` (string | array | object):
  - Array: Check if column array contains all elements
  - Object: Check if column JSON contains all key-value pairs
  - String: For range types (e.g., `'[1,10]'`)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Array contains elements
const { data } = await client
  .from('posts')
  .select()
  .contains('tags', ['javascript', 'typescript'])
// Matches rows where tags includes BOTH 'javascript' AND 'typescript'

// JSON contains key-value pairs
const { data } = await client
  .from('users')
  .select()
  .contains('metadata', { premium: true, verified: true })
// Matches rows where metadata has both keys with matching values

// Range contains range
const { data } = await client
  .from('schedules')
  .select()
  .contains('time_range', '[9:00,10:00]')
```

**PostgREST Mapping:**
- URL param (array): `?tags=cs.{javascript,typescript}`
- URL param (json): `?metadata=cs.{"premium":true,"verified":true}`
- URL param (range): `?time_range=cs.[9:00,10:00]`
- Operator: `cs` = "contains" (Postgres `@>`)

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial (requires JSON extension)
- **SQL (array as JSON):** Use json_each + checking
- **SQL (JSON):**
  ```sql
  WHERE json_extract(metadata, '$.premium') = true
    AND json_extract(metadata, '$.verified') = true
  ```
- **Notes:**
  - SQLite has no native array type (use JSON array)
  - JSON operations require json1 extension (built-in since 3.38)
  - Range types: No native support
  - For arrays: Store as JSON, iterate with json_each()
- **Workarounds:**
  - Arrays: Store as JSON text: `'["javascript","typescript"]'`
  - Check containment with json_each:
    ```sql
    WHERE EXISTS (
      SELECT 1 FROM json_each(tags)
      WHERE value IN ('javascript', 'typescript')
      HAVING COUNT(*) = 2
    )
    ```
  - JSON objects: Extract each key with json_extract
  - Range: Store as text, parse manually

**Edge Cases:**
- Empty array/object: Matches all rows (vacuously true)
- NULL in array: Postgres handles, SQLite needs special care
- Nested JSON: Requires recursive checking
- Partial match: `contains()` is strict (all must be present)

**Related:** containedBy(), overlaps()

**Source:** PostgrestFilterBuilder.ts:410-435

---

## `containedBy(column, value)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
containedBy<ColumnName extends string & keyof Row>(
  column: ColumnName,
  value: string | ReadonlyArray<Row[ColumnName]> | Record<string, unknown>
): this
```

**Purpose:** Match rows where every element in column is contained by value. Inverse of `contains()`.

**Parameters:**
- `column` (string) - Column name (must be array, jsonb, or range type)
- `value` (string | array | object):
  - Array: Check if all column elements are in value array
  - Object: Check if all column JSON keys are in value
  - String: For range types

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Array contained by larger array
const { data } = await client
  .from('posts')
  .select()
  .containedBy('tags', ['javascript', 'typescript', 'python', 'rust'])
// Matches rows where tags only has values from the provided list

// JSON contained by larger object
const { data } = await client
  .from('users')
  .select()
  .containedBy('settings', { theme: 'dark', lang: 'en', timezone: 'UTC' })
// Matches rows where settings has subset of provided keys

// Range contained by range
const { data } = await client
  .from('schedules')
  .select()
  .containedBy('time_range', '[8:00,18:00]')
```

**PostgREST Mapping:**
- URL param (array): `?tags=cd.{javascript,typescript,python,rust}`
- URL param (json): `?settings=cd.{"theme":"dark","lang":"en","timezone":"UTC"}`
- Operator: `cd` = "contained by" (Postgres `<@`)

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial
- **SQL (array as JSON):**
  ```sql
  WHERE NOT EXISTS (
    SELECT 1 FROM json_each(tags)
    WHERE value NOT IN ('javascript', 'typescript', 'python', 'rust')
  )
  ```
- **Notes:**
  - Same limitations as `contains()`
  - Requires checking that no elements fall outside value set
- **Workarounds:**
  - Similar to `contains()` but inverse logic
  - Check all column elements are in value set

**Edge Cases:**
- Empty column array: Always contained (matches all)
- Empty value array: Only matches empty column arrays
- Superset/subset relationship

**Related:** contains(), overlaps()

**Source:** PostgrestFilterBuilder.ts:437-461

---

## `overlaps(column, value)`

**Priority:** Medium | **SQLite:** ⚠️ Partial

**Signature:**
```ts
overlaps<ColumnName extends string & keyof Row>(
  column: ColumnName,
  value: string | ReadonlyArray<Row[ColumnName]>
): this
```

**Purpose:** Match rows where column and value have at least one element in common. For arrays and ranges.

**Parameters:**
- `column` (string) - Column name (must be array or range type)
- `value` (string | array):
  - Array: Check if column array shares any element
  - String: For range types (overlap check)

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Arrays with common elements
const { data } = await client
  .from('posts')
  .select()
  .overlaps('tags', ['javascript', 'python'])
// Matches rows where tags includes 'javascript' OR 'python' (or both)

// Range overlaps
const { data } = await client
  .from('schedules')
  .select()
  .overlaps('time_range', '[9:00,12:00]')
// Matches rows where time_range overlaps with 9:00-12:00
```

**PostgREST Mapping:**
- URL param (array): `?tags=ov.{javascript,python}`
- URL param (range): `?time_range=ov.[9:00,12:00]`
- Operator: `ov` = "overlaps" (Postgres `&&`)

**SQLite Compatibility (Theoretical):**
- **Status:** ⚠️ Partial
- **SQL (array as JSON):**
  ```sql
  WHERE EXISTS (
    SELECT 1 FROM json_each(tags)
    WHERE value IN ('javascript', 'python')
  )
  ```
- **Notes:**
  - Check if any column element is in value set
  - For ranges: Parse and compare boundaries
- **Workarounds:**
  - Arrays: Use json_each with IN check
  - Ranges: Manual overlap logic:
    ```sql
    WHERE NOT (range_end < '9:00' OR range_start > '12:00')
    ```

**Edge Cases:**
- Empty arrays: No overlap (matches nothing)
- Single element: Same as checking if value contains that element
- Ranges: Overlap includes touching boundaries

**Related:** contains(), containedBy(), in()

**Source:** PostgrestFilterBuilder.ts:536-557

---

## Summary

Array/JSON filters with SQLite compatibility:

| Method | PostgREST | SQLite | Complexity |
|--------|-----------|--------|-----------|
| `contains()` | `cs.{...}` or `cs.{...}` | JSON + json_each | ⚠️ Requires JSON extension |
| `containedBy()` | `cd.{...}` or `cd.{...}` | JSON + json_each | ⚠️ Requires JSON extension |
| `overlaps()` | `ov.{...}` | JSON + json_each | ⚠️ Requires JSON extension |

**SQLite Implementation Strategy:**
1. Store arrays as JSON text: `'["a","b","c"]'`
2. Use json1 extension (built-in since SQLite 3.38)
3. Leverage `json_each()` table-valued function
4. For objects, use `json_extract(column, '$.key')`

**Limitations:**
- No native array type in SQLite
- Range types not supported (use separate start/end columns)
- Performance: JSON operations slower than native arrays
- Need explicit JSON validation/constraints
