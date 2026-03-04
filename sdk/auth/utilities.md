# Utilities

**Priority:** LOW
**Methods:** 2
**Status:** Complete

JWT verification and helper utilities.

---

## Methods (Brief Format)

#### `getClaims(jwt?, options?)`
**Priority:** Low | **Complexity:** Moderate

**Signature:**
```typescript
async getClaims(
  jwt?: string,
  options?: {
    keys?: JWK[]           // @deprecated - use options.jwks
    allowExpired?: boolean  // skip exp claim validation
    jwks?: { keys: JWK[] } // override cached JWKS
  }
): Promise<
  | { data: { claims: JwtPayload; header: JwtHeader; signature: Uint8Array }; error: null }
  | { data: null; error: AuthError }
  | { data: null; error: null }
>
```
**Purpose:** Verify JWT offline via JWKS endpoint (`/.well-known/jwks.json`), extract claims without hitting Auth server. Falls back to server verification if project uses symmetric (HS256) signing.
**Usage:**
```typescript
// verify current session token
const { data, error } = await supabase.auth.getClaims()
console.log(data.claims.sub, data.claims.role)

// verify specific JWT with options
const { data } = await supabase.auth.getClaims(accessToken, { allowExpired: true })
```
**Implementation:** Requires JWKS endpoint serving project's public keys. Caches JWKS globally across client instances (useful for serverless/edge). Asymmetric keys (RS256/ES256) verified locally; HS256 falls back to `getUser()` server call. Parse JWT header to extract `kid`, fetch matching JWK, verify signature.
**Use Case:** High-throughput token verification without per-request Auth server calls - edge functions, middleware, API routes.

---

#### `isThrowOnErrorEnabled()`
**Priority:** Low | **Complexity:** Simple

**Signature:**
```typescript
public isThrowOnErrorEnabled(): boolean
```
**Purpose:** Check if client is configured to throw errors instead of returning them in response objects.
**Usage:**
```typescript
const throws = supabase.auth.isThrowOnErrorEnabled()
// true = errors thrown as exceptions
// false = errors returned as { data: null, error: AuthError }
```
**Implementation:** Returns internal `throwOnError` boolean set via `GoTrueClientOptions`. No async, no side effects.
**Use Case:** Conditional error handling logic, library wrappers that need to adapt to client's error mode.

---

## Summary

| Method | Purpose | Complexity |
|---|---|---|
| `getClaims` | Offline JWT verification via JWKS | Moderate |
| `isThrowOnErrorEnabled` | Check error throwing mode | Simple |

**Key implementation notes:**
- `getClaims` is the preferred alternative to `getUser` for token verification - avoids per-request server roundtrip when using asymmetric JWTs
- JWKS cache is global (shared across all `GoTrueClient` instances with same storage key) - optimized for serverless environments
- `throwOnError` mode changes all auth methods from returning errors to throwing them
