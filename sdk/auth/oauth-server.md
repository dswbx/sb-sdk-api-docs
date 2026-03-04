# OAuth 2.1 Server

**Priority:** LOW
**Methods:** 5
**Status:** Complete

OAuth 2.1 authorization server methods for custom OAuth flows. Used to implement the consent page when Supabase Auth acts as an OAuth 2.1 authorization server. All methods require an authenticated session.

---

## Methods (Brief Format)

#### `oauth.getAuthorizationDetails(authorizationId)`
**Priority:** Low | **Complexity:** Moderate

**Signature:**
```typescript
getAuthorizationDetails(authorizationId: string): Promise<AuthOAuthAuthorizationDetailsResponse>
```
**Purpose:** Retrieve details about an OAuth authorization request to display on a consent page.
**Usage:**
```typescript
const { data, error } = await supabase.auth.oauth.getAuthorizationDetails(authorizationId)
// data: { authorization_id, client, user, scope, redirect_url? }
```
**Implementation:** GET to `/oauth/authorizations/{id}`. If response includes `redirect_url`, consent was already given - handle redirect. Requires session.
**Use Case:** Building a custom consent page that shows which client is requesting access and what scopes are requested.

---

#### `oauth.approveAuthorization(authorizationId, options?)`
**Priority:** Low | **Complexity:** Moderate

**Signature:**
```typescript
approveAuthorization(
  authorizationId: string,
  options?: { skipBrowserRedirect?: boolean }
): Promise<AuthOAuthConsentResponse>
```
**Purpose:** Approve an OAuth authorization request, granting the client access.
**Usage:**
```typescript
const { data, error } = await supabase.auth.oauth.approveAuthorization(authorizationId)
// data: { redirect_url } — browser auto-redirects unless skipBrowserRedirect is true
```
**Implementation:** POST to `/oauth/authorizations/{id}/consent` with `{ action: 'approve' }`. Auto-redirects in browser by default; set `skipBrowserRedirect: true` to handle redirect manually (e.g., server-side).
**Use Case:** User clicks "Allow" on consent page to grant an OAuth client access to their account.

---

#### `oauth.denyAuthorization(authorizationId, options?)`
**Priority:** Low | **Complexity:** Moderate

**Signature:**
```typescript
denyAuthorization(
  authorizationId: string,
  options?: { skipBrowserRedirect?: boolean }
): Promise<AuthOAuthConsentResponse>
```
**Purpose:** Deny an OAuth authorization request, refusing the client access.
**Usage:**
```typescript
const { data, error } = await supabase.auth.oauth.denyAuthorization(authorizationId)
// data: { redirect_url } — browser auto-redirects unless skipBrowserRedirect is true
```
**Implementation:** POST to `/oauth/authorizations/{id}/consent` with `{ action: 'deny' }`. Same redirect behavior as `approveAuthorization`.
**Use Case:** User clicks "Deny" on consent page to reject an OAuth client's access request.

---

#### `oauth.listGrants()`
**Priority:** Low | **Complexity:** Simple

**Signature:**
```typescript
listGrants(): Promise<AuthOAuthGrantsResponse>
```
**Purpose:** List all OAuth grants the authenticated user has authorized.
**Usage:**
```typescript
const { data, error } = await supabase.auth.oauth.listGrants()
// data: OAuthGrant[] — each grant has { client, scopes, granted_at }
```
**Implementation:** GET to `/user/oauth/grants`. Returns array of grants with client info (`id`, `name`, `uri`, `logo_uri`), granted scopes, and timestamp.
**Use Case:** Building an "authorized applications" settings page where users can review which apps have access.

---

#### `oauth.revokeGrant(options)`
**Priority:** Low | **Complexity:** Moderate

**Signature:**
```typescript
revokeGrant(options: { clientId: string }): Promise<AuthOAuthRevokeGrantResponse>
```
**Purpose:** Revoke a user's OAuth grant for a specific client.
**Usage:**
```typescript
const { data, error } = await supabase.auth.oauth.revokeGrant({ clientId: 'uuid' })
// data: {} on success
```
**Implementation:** DELETE to `/user/oauth/grants?client_id={id}`. Marks consent as revoked, deletes active sessions for that client, and invalidates associated refresh tokens. Destructive operation.
**Use Case:** User revokes an app's access from their "authorized applications" settings page.

---

## Key Types

```typescript
type OAuthAuthorizationDetails = {
  authorization_id: string
  redirect_url?: string            // present if already consented
  client: OAuthAuthorizationClient // { id, name, uri, logo_uri }
  user: { id: string; email: string }
  scope: string                    // space-separated scopes
}

type OAuthGrant = {
  client: OAuthAuthorizationClient
  scopes: string[]
  granted_at: string               // ISO 8601
}
```

---

## Summary

| Method | Complexity | HTTP | Endpoint |
|--------|-----------|------|----------|
| `getAuthorizationDetails` | Moderate | GET | `/oauth/authorizations/{id}` |
| `approveAuthorization` | Moderate | POST | `/oauth/authorizations/{id}/consent` |
| `denyAuthorization` | Moderate | POST | `/oauth/authorizations/{id}/consent` |
| `listGrants` | Simple | GET | `/user/oauth/grants` |
| `revokeGrant` | Moderate | DELETE | `/user/oauth/grants` |

All methods require an authenticated session. The consent methods (`getAuthorizationDetails`, `approveAuthorization`, `denyAuthorization`) are used together to build a consent page. The grant methods (`listGrants`, `revokeGrant`) are used together to build an authorized-applications settings page.
