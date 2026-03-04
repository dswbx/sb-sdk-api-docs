# Identity Linking

**Priority:** MEDIUM
**Methods:** 2
**Status:** Complete

OAuth/OIDC identity linking for connecting multiple authentication providers to a single account.

---

## Methods

### `linkIdentity(credentials)`

**Priority:** Medium | **Complexity:** Moderate

**Signature:**

```ts
// OAuth flow (redirect-based)
async linkIdentity(credentials: SignInWithOAuthCredentials): Promise<OAuthResponse>

// ID token flow (server-side / native)
async linkIdentity(credentials: SignInWithIdTokenCredentials): Promise<AuthTokenResponse>
```

**Purpose:** Link an additional OAuth/OIDC identity to the current user's account

**Parameters:**

**OAuth flow:**
- `credentials.provider` (Provider) - OAuth provider ('google', 'github', 'apple', etc.)
- `credentials.options.redirectTo?` (string) - URL to redirect after OAuth consent
- `credentials.options.scopes?` (string) - Space-separated OAuth scopes
- `credentials.options.queryParams?` (object) - Additional query parameters for provider
- `credentials.options.skipBrowserRedirect?` (boolean) - If true, return URL without redirecting

**ID token flow:**
- `credentials.provider` (Provider) - OIDC provider
- `credentials.token` (string) - OIDC ID token
- `credentials.access_token?` (string) - Provider access token (for `at_hash` validation)
- `credentials.nonce?` (string) - Nonce for replay protection
- `credentials.options.captchaToken?` (string) - CAPTCHA token

**Returns:**

```ts
// OAuth flow
{ data: { provider: string; url: string | null }, error: AuthError | null }

// ID token flow
{ data: { user: User; session: Session }, error: AuthError | null }
```

**Usage:**

```ts
// Link Google account via OAuth redirect
const { data, error } = await client.auth.linkIdentity({
  provider: 'google',
})
// Browser redirects to Google consent screen
// After consent, user returns with linked identity

// Link with redirect URL
const { data, error } = await client.auth.linkIdentity({
  provider: 'github',
  options: {
    redirectTo: 'https://myapp.com/settings/accounts',
  },
})

// Get URL without redirecting
const { data, error } = await client.auth.linkIdentity({
  provider: 'google',
  options: { skipBrowserRedirect: true },
})
console.log('Link URL:', data.url)

// Link via ID token (native apps, server-side)
const { data, error } = await client.auth.linkIdentity({
  provider: 'google',
  token: googleIdToken,
  nonce: 'random_nonce',
})
// Session updated immediately, no redirect needed
```

**Implementation Considerations:**

- **Backend Requirements:** OAuth provider config, identity storage, user identity table
- **Security:** Requires active session (JWT). OAuth flow supports PKCE. ID token flow validates token server-side.
- **Complexity:** Moderate - two distinct flows dispatched based on `'token' in credentials`
- **Dependencies:** OAuth provider SDKs (OAuth flow), OIDC token validation (ID token flow)
- **Storage:** ID token flow saves updated session and emits `USER_UPDATED` event

**Authentication Flow:**

```
# OAuth flow
linkIdentity({ provider: 'google' })
  → _useSession() to get current JWT
  → GET /user/identities/authorize?provider=google
      → Headers: Authorization: Bearer <jwt>
      → Query: provider, redirect_to, scopes
  → Returns { url } - provider authorization URL
  → Browser redirects to provider consent screen
  → Provider redirects back to app
  → initialize() detects callback, links identity
  → USER_UPDATED event emitted

# ID token flow
linkIdentity({ provider: 'google', token: idToken })
  → _useSession() to get current JWT
  → POST /token?grant_type=id_token
      → Headers: Authorization: Bearer <jwt>
      → Body: { provider, id_token, access_token, nonce, link_identity: true }
  → Backend validates ID token, links identity to user
  → Returns updated session
  → _saveSession() + emit USER_UPDATED
```

**GoTrue Backend Mapping:**

**OAuth flow:**
- **Endpoint:** `GET /user/identities/authorize`
- **Headers:** `Authorization: Bearer <jwt>`
- **Query:** `provider=google&redirect_to=...&scopes=...`
- **Response:** `{ url: "https://accounts.google.com/o/oauth2/..." }`

**ID token flow:**
- **Endpoint:** `POST /token?grant_type=id_token`
- **Headers:** `Authorization: Bearer <jwt>`
- **Request Body:**

```json
{
  "provider": "google",
  "id_token": "eyJ...",
  "access_token": "ya29...",
  "nonce": "random_nonce",
  "link_identity": true,
  "gotrue_meta_security": { "captcha_token": "..." }
}
```

- **Response:** Standard session response `{ access_token, refresh_token, user }`

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `identity_already_exists` - identity already linked to another user
- `single_identity_not_deletable` - cannot link if it would create conflict
- `AuthInvalidTokenResponseError` - invalid ID token response

**Related:** `unlinkIdentity`, `getUserIdentities`, `signInWithOAuth`, `signInWithIdToken`

---

### `unlinkIdentity(identity)`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async unlinkIdentity(identity: UserIdentity): Promise<
  | { data: {}; error: null }
  | { data: null; error: AuthError }
>
```

**Purpose:** Remove a linked identity from user account; user can no longer sign in with that provider

**Parameters:**

- `identity` (UserIdentity) - The identity object to unlink (from `getUserIdentities()`)
  - Uses `identity.identity_id` (not `identity.id`)

**Returns:**

```ts
{ data: {}, error: AuthError | null }
```

**Usage:**

```ts
// Get identities first
const { data: { identities } } = await client.auth.getUserIdentities()

// Find and unlink GitHub identity
const githubIdentity = identities.find(i => i.provider === 'github')

if (githubIdentity) {
  const { error } = await client.auth.unlinkIdentity(githubIdentity)
  if (!error) {
    console.log('GitHub identity unlinked')
  }
}

// Unlink with safety check (must keep at least one identity)
const { data: { identities } } = await client.auth.getUserIdentities()

if (identities.length > 1) {
  const target = identities.find(i => i.provider === 'google')
  if (target) {
    await client.auth.unlinkIdentity(target)
  }
} else {
  console.log('Cannot unlink last identity')
}
```

**Implementation Considerations:**

- **Backend Requirements:** Identity deletion endpoint, constraint to prevent unlinking last identity
- **Security:** Requires active session. Backend should verify user owns the identity.
- **Complexity:** Simple - single DELETE request
- **Dependencies:** None
- **Storage:** No client-side storage operations

**Authentication Flow:**

```
unlinkIdentity(identity)
  → _useSession() to get current JWT
  → DELETE /user/identities/{identity_id}
      → Headers: Authorization: Bearer <jwt>
  → Backend:
      → Verify user owns identity
      → Check user has other identities (prevent lockout)
      → Delete identity record
  → Return success
```

**GoTrue Backend Mapping:**

- **Endpoint:** `DELETE /user/identities/{identity_id}`
- **Headers:** `Authorization: Bearer <jwt>`
- **Response:** 200 OK (empty body)

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `single_identity_not_deletable` - cannot unlink last identity (would lock user out)
- `identity_not_found` - identity doesn't exist or doesn't belong to user
- `unauthorized` - invalid JWT

**Related:** `linkIdentity`, `getUserIdentities`

---

## Summary

**2 identity linking methods documented:**

- ✅ linkIdentity - Link OAuth/OIDC identity (two flows: OAuth redirect + ID token)
- ✅ unlinkIdentity - Remove linked identity

**Key Patterns:**

1. **Overloaded method:** `linkIdentity` dispatches based on `'token' in credentials` - OAuth flow for browser redirect, ID token flow for native/server-side
2. **Session required:** Both methods require active session
3. **Safety constraint:** Cannot unlink last identity (backend enforces)
4. **Event emission:** ID token flow emits `USER_UPDATED`; OAuth flow emits after callback completes
5. **PKCE support:** OAuth flow supports PKCE code challenge

**Common Use Cases:**

- **Account linking page:** `getUserIdentities` → show linked providers → `linkIdentity`/`unlinkIdentity`
- **Social login upgrade:** Anonymous user → `linkIdentity({ provider: 'google' })`
- **Native app linking:** `linkIdentity({ provider: 'apple', token: appleIdToken })`
