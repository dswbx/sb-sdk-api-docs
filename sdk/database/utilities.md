# Utilities

**Category:** MEDIUM Priority | **Status:** ✅ Complete
**Methods:** 3 (abortSignal, setHeader, throwOnError)

Utility methods for request control, custom headers, and error handling.

---

## `abortSignal(signal)`

**Priority:** Medium | **SQLite:** ✅ Full

**Signature:**
```ts
abortSignal(signal: AbortSignal): this
```

**Purpose:** Set AbortSignal for request cancellation.

**Parameters:**
- `signal` (AbortSignal) - AbortSignal from AbortController for canceling request

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Basic cancellation
const controller = new AbortController()
const signal = controller.signal

const promise = client
  .from('users')
  .select('*')
  .abortSignal(signal)

// Cancel after timeout
setTimeout(() => controller.abort(), 5000)

try {
  const { data } = await promise
} catch (error) {
  if (error.name === 'AbortError') {
    console.log('Request cancelled')
  }
}

// React component cleanup
useEffect(() => {
  const controller = new AbortController()

  client
    .from('posts')
    .select('*')
    .abortSignal(controller.signal)
    .then(({ data }) => setPosts(data))

  return () => controller.abort()
}, [])

// Timeout helper
function withTimeout(ms) {
  const controller = new AbortController()
  setTimeout(() => controller.abort(), ms)
  return controller.signal
}

const { data } = await client
  .from('users')
  .select('*')
  .abortSignal(withTimeout(10000))
```

**PostgREST Mapping:**
- Passed to fetch() call
- Standard browser/Node.js fetch AbortSignal
- Aborts HTTP request in flight

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full (if async operations supported)
- **Notes:**
  - Local SQLite operations typically fast (< ms)
  - Useful for long-running queries or remote SQLite (over network)
  - Implement by checking signal.aborted before/during query
- **Implementation:**
  ```ts
  async function executeQuery(sql, params, signal) {
    // Check before starting
    if (signal?.aborted) {
      throw new DOMException('Aborted', 'AbortError')
    }

    // For long queries, check periodically
    const result = await db.execute(sql, params)

    // Check after
    if (signal?.aborted) {
      throw new DOMException('Aborted', 'AbortError')
    }

    return result
  }
  ```
- **Workarounds:**
  - For synchronous SQLite: Limited cancellation support
  - For async SQLite: Check signal.aborted at query boundaries
  - Some SQLite wrappers support interrupting queries

**Edge Cases:**
- Signal already aborted: Query fails immediately
- Signal aborted during fetch: Throws AbortError
- Multiple signals: Last one wins
- Signal without abort: Query proceeds normally

**Related:** throwOnError(), setHeader()

**Source:** PostgrestTransformBuilder.ts:195-203

---

## `setHeader(name, value)`

**Priority:** Medium | **SQLite:** ✅ Full (if HTTP layer exists)

**Signature:**
```ts
setHeader(name: string, value: string): this
```

**Purpose:** Set custom HTTP header for request.

**Parameters:**
- `name` (string) - Header name
- `value` (string) - Header value

**Returns:** Same builder for chaining.

**Usage:**
```ts
// Custom auth header
const { data } = await client
  .from('users')
  .select('*')
  .setHeader('X-Custom-Token', 'abc123')

// Request ID for tracing
const { data } = await client
  .from('orders')
  .select('*')
  .setHeader('X-Request-ID', crypto.randomUUID())

// Custom Prefer header (override default)
const { data } = await client
  .from('products')
  .insert({ name: 'Widget' })
  .setHeader('Prefer', 'resolution=merge-duplicates,return=representation')
  .select()

// Multiple headers
const { data } = await client
  .from('posts')
  .select('*')
  .setHeader('X-API-Version', '2')
  .setHeader('X-Client-ID', 'web-app')
```

**PostgREST Mapping:**
- Adds header to HTTP request
- Can override default headers
- Useful for:
  - Custom authentication
  - API versioning
  - Request tracing
  - Feature flags

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full (if applicable)
- **Notes:**
  - Local SQLite: No HTTP headers (N/A)
  - Remote SQLite over HTTP: Pass headers through
  - Custom context: Store in query context object
- **Implementation:**
  ```ts
  // For HTTP-based SQLite
  async function executeQuery(sql, params, headers) {
    return fetch('/sqlite-api', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...headers
      },
      body: JSON.stringify({ sql, params })
    })
  }

  // For local SQLite, use context
  const context = {
    headers: new Map(),
    ...
  }
  context.headers.set('X-Request-ID', '123')
  ```

**Edge Cases:**
- Header name: Case-insensitive
- Duplicate names: Last value wins (overwrites)
- Reserved headers: Some may be ignored/overridden by client
- Empty value: Typically removes header

**Related:** abortSignal(), throwOnError()

**Source:** PostgrestBuilder.ts:82-89

---

## `throwOnError()`

**Priority:** Medium | **SQLite:** ✅ Full

**Signature:**
```ts
throwOnError(): this & PostgrestBuilder<Result, true>
```

**Purpose:** Change error handling from returned error to thrown exception.

**Parameters:** None

**Returns:** Same builder with type indicating errors will be thrown.

**Usage:**
```ts
// Default behavior (returned error)
const { data, error } = await client
  .from('users')
  .select('*')

if (error) {
  console.error('Query failed:', error)
}

// With throwOnError (thrown exception)
try {
  const { data } = await client
    .from('users')
    .select('*')
    .throwOnError()
  // No need to check error, it's guaranteed null here
  console.log(data)
} catch (error) {
  console.error('Query failed:', error)
  // error is PostgrestError instance
}

// Use with async/await pattern
async function getUser(id) {
  const { data } = await client
    .from('users')
    .select('*')
    .eq('id', id)
    .single()
    .throwOnError()
  return data // Type: User (not User | null)
}

// Cleaner error boundaries
try {
  const users = await getUsers()
  const products = await getProducts()
  const orders = await getOrders()
  // All succeeded
} catch (error) {
  // Any failed, handle in one place
}
```

**PostgREST Mapping:**
- No HTTP change
- Changes response handling:
  - Default: `{ data, error }` (error is null or object)
  - With throwOnError: Throws PostgrestError if error exists

**SQLite Compatibility (Theoretical):**
- **Status:** ✅ Full
- **Notes:**
  - Error handling pattern choice
  - Default: Return `{ data, error }` tuple
  - With throwOnError: Throw exception on error
- **Implementation:**
  ```ts
  class QueryBuilder {
    shouldThrow = false

    throwOnError() {
      this.shouldThrow = true
      return this
    }

    async execute() {
      try {
        const data = await executeQuery(this.sql, this.params)
        return { data, error: null }
      } catch (error) {
        if (this.shouldThrow) {
          throw error
        }
        return { data: null, error }
      }
    }
  }
  ```

**Edge Cases:**
- Type narrowing: TypeScript knows data is not null in success case
- Error type: PostgrestError (wraps database error)
- Fetch errors: Also thrown (network, timeout, etc.)
- Default is returned error (for Go-style error handling)

**Related:** abortSignal(), setHeader()

**Source:** PostgrestBuilder.ts:71-80

---

## Summary

Utility methods with SQLite compatibility:

| Method | Purpose | SQLite | Notes |
|--------|---------|--------|-------|
| `abortSignal()` | Cancel request | ✅ Full | Check signal.aborted during query |
| `setHeader()` | Custom HTTP headers | ✅ Full (if HTTP) | N/A for local SQLite |
| `throwOnError()` | Exception-based errors | ✅ Full | Error handling pattern |

**Implementation Notes:**
- `abortSignal()`: Most useful for network requests; limited value for local SQLite
- `setHeader()`: Only relevant if SQLite accessed over HTTP
- `throwOnError()`: Pure JS error handling pattern, works anywhere

**Best Practices:**
- Use `abortSignal()` in React components with cleanup
- Use `throwOnError()` for cleaner async/await code
- Use `setHeader()` for tracing and custom auth
