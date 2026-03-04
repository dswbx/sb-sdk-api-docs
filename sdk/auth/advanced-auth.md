# Advanced Authentication

**Priority:** MEDIUM
**Methods:** 6
**Status:** Complete

Enterprise auth methods including SSO, ID token authentication, Web3, anonymous auth, and auto-refresh.

---

## Methods

### `signInWithSSO(params)`

**Priority:** Medium | **Complexity:** Complex

**Signature:**

```ts
async signInWithSSO(params: SignInWithSSO): Promise<SSOResponse>

type SignInWithSSO =
  | {
      providerId: string
      options?: {
        redirectTo?: string
        captchaToken?: string
        skipBrowserRedirect?: boolean
      }
    }
  | {
      domain: string
      options?: {
        redirectTo?: string
        captchaToken?: string
        skipBrowserRedirect?: boolean
      }
    }

type SSOResponse = {
  data: { url: string } | null
  error: AuthError | null
}
```

**Purpose:** Initiate enterprise single-sign-on via SAML identity provider

**Parameters:**

- `params.providerId` (string) - UUID of the SSO identity provider (mutually exclusive with `domain`)
- `params.domain` (string) - Domain of the organization (e.g. `acme.com`) (mutually exclusive with `providerId`)
- `params.options.redirectTo?` (string) - URL to redirect after sign-in
- `params.options.captchaToken?` (string) - CAPTCHA verification token
- `params.options.skipBrowserRedirect?` (boolean) - If true, don't auto-redirect browser. Default: false

**Returns:**

```ts
{ data: { url: string }, error: null }
// or
{ data: null, error: AuthError }
```

**Usage:**

```ts
// SSO by domain (extract from user's email)
const { data, error } = await client.auth.signInWithSSO({
  domain: 'acme.com',
})
// Browser auto-redirects to IdP login page

// SSO by provider UUID (org-specific login page)
const { data, error } = await client.auth.signInWithSSO({
  providerId: '3fa85f64-5717-4562-b3fc-2c963f66afa6',
})

// Manual redirect handling
const { data, error } = await client.auth.signInWithSSO({
  domain: 'acme.com',
  options: {
    skipBrowserRedirect: true,
    redirectTo: 'https://myapp.com/auth/callback',
  },
})
if (data?.url) {
  // Handle redirect manually
  window.location.href = data.url
}

// With CAPTCHA
const { data, error } = await client.auth.signInWithSSO({
  domain: 'acme.com',
  options: {
    captchaToken: 'captcha_token...',
  },
})
```

**Implementation Considerations:**

- **Backend Requirements:** SSO/SAML provider configuration, IdP metadata registration, redirect URL handling
- **Security:** PKCE code challenge generated automatically when flowType is `pkce`; `skip_http_redirect` always sent as `true` (fetch can't follow redirects)
- **Complexity:** Complex - requires SAML IdP setup, metadata exchange, assertion parsing
- **Dependencies:** SAML identity provider, GoTrue SSO endpoints, organization/domain registry
- **Storage:** Code verifier stored for PKCE flow; cleaned up on error

**Authentication Flow:**

```
signInWithSSO({ domain | providerId })
  → If PKCE: generate code_challenge + store code_verifier
  → POST /sso
      → Body: { domain|provider_id, redirect_to, skip_http_redirect: true,
                code_challenge, code_challenge_method, gotrue_meta_security }
  → Backend:
      → Lookup SSO provider by domain or provider_id
      → Generate SAML AuthnRequest
      → Return redirect URL to IdP
  → Client:
      → If browser and !skipBrowserRedirect: window.location.assign(url)
      → Return { data: { url }, error: null }
  → User authenticates at IdP
  → IdP redirects back to GoTrue callback
  → GoTrue validates SAML assertion
  → Redirects to app with auth code/token
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /sso`
- **Request Body:**

```json
{
  "provider_id": "uuid",
  "domain": "acme.com",
  "redirect_to": "https://myapp.com/callback",
  "skip_http_redirect": true,
  "code_challenge": "...",
  "code_challenge_method": "S256",
  "gotrue_meta_security": { "captcha_token": "..." }
}
```

- **Response:**

```json
{
  "url": "https://idp.acme.com/saml/sso?SAMLRequest=..."
}
```

**Error Cases:**

- `not_found` - SSO provider not found for domain/provider_id
- `validation_failed` - invalid domain or provider_id format
- PKCE code verifier cleaned up from storage on error

**Related:** `signInWithOAuth`, `exchangeCodeForSession`

---

### `signInWithIdToken(credentials)`

**Priority:** Medium | **Complexity:** Moderate

**Signature:**

```ts
async signInWithIdToken(
  credentials: SignInWithIdTokenCredentials
): Promise<AuthTokenResponse>

type SignInWithIdTokenCredentials = {
  provider: 'google' | 'apple' | 'azure' | 'facebook' | 'kakao' | (string & {})
  token: string
  access_token?: string
  nonce?: string
  options?: {
    captchaToken?: string
  }
}
```

**Purpose:** Sign in using an OIDC ID token from an external provider

**Parameters:**

- `credentials.provider` (string) - Provider name or OIDC `iss` value. Supported: `google`, `apple`, `azure`, `facebook`, `kakao`
- `credentials.token` (string) - OIDC ID token from the provider. `iss` claim must match provider
- `credentials.access_token?` (string) - Required if ID token contains `at_hash` claim
- `credentials.nonce?` (string) - Required if ID token contains `nonce` claim
- `credentials.options.captchaToken?` (string) - CAPTCHA verification token

**Returns:**

```ts
{ data: { user: User, session: Session }, error: null }
// or
{ data: { user: null, session: null }, error: AuthError }
```

**Usage:**

```ts
// Google ID token (e.g. from Google Sign-In SDK)
const { data, error } = await client.auth.signInWithIdToken({
  provider: 'google',
  token: googleIdToken,
})

// Apple ID token with nonce
const { data, error } = await client.auth.signInWithIdToken({
  provider: 'apple',
  token: appleIdToken,
  nonce: 'random_nonce_used_during_apple_auth',
})

// With access_token (when ID token has at_hash claim)
const { data, error } = await client.auth.signInWithIdToken({
  provider: 'google',
  token: googleIdToken,
  access_token: googleAccessToken,
})

// React Native / mobile flow
// 1. Get ID token from native SDK (Google Sign-In, Apple Auth)
// 2. Exchange for Supabase session
const { data, error } = await client.auth.signInWithIdToken({
  provider: 'google',
  token: nativeGoogleIdToken,
})
if (data.session) {
  // User is signed in
}
```

**Implementation Considerations:**

- **Backend Requirements:** OIDC provider configuration (client ID, issuer URL), token verification (signature, claims)
- **Security:** Backend must verify ID token signature against provider's JWKS, validate `iss`, `aud`, `exp` claims; nonce prevents replay attacks
- **Complexity:** Moderate - token verification is backend-side; client just passes token
- **Dependencies:** OIDC provider SDK for obtaining ID token (Google Sign-In, Apple Auth, etc.)
- **Storage:** Session saved to storage, SIGNED_IN event emitted

**Authentication Flow:**

```
signInWithIdToken({ provider, token, access_token?, nonce? })
  → POST /token?grant_type=id_token
      → Body: { provider, id_token, access_token, nonce, gotrue_meta_security }
  → Backend:
      → Fetch provider's JWKS (cached)
      → Verify ID token signature
      → Validate claims (iss, aud, exp, nonce, at_hash)
      → Find or create user by provider + sub claim
      → Issue session (access_token + refresh_token)
  → Client:
      → Save session to storage
      → Notify subscribers (SIGNED_IN)
      → Return { data: { user, session }, error: null }
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /token?grant_type=id_token`
- **Request Body:**

```json
{
  "provider": "google",
  "id_token": "eyJhbGciOi...",
  "access_token": "ya29...",
  "nonce": "random_nonce",
  "gotrue_meta_security": { "captcha_token": "..." }
}
```

- **Response:**

```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "bearer",
  "expires_in": 3600,
  "expires_at": 1700000000,
  "refresh_token": "...",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "app_metadata": { "provider": "google" },
    ...
  }
}
```

**Error Cases:**

- `invalid_grant` - ID token invalid, expired, or signature verification failed
- `AuthInvalidTokenResponseError` - response missing session or user
- `provider_not_found` - provider not configured on GoTrue instance
- `nonce_mismatch` - nonce in token doesn't match provided nonce

**Related:** `signInWithOAuth`, `signInWithSSO`

---

### `signInWithWeb3(credentials)`

**Priority:** Medium | **Complexity:** Complex

**Signature:**

```ts
async signInWithWeb3(credentials: Web3Credentials): Promise<
  | { data: { session: Session; user: User }; error: null }
  | { data: { session: null; user: null }; error: AuthError }
>

type Web3Credentials = SolanaWeb3Credentials | EthereumWeb3Credentials

type EthereumWeb3Credentials =
  | {
      chain: 'ethereum'
      wallet?: EthereumWallet  // EIP-1193 provider, defaults to window.ethereum
      statement?: string
      options?: {
        url?: string
        captchaToken?: string
        signInWithEthereum?: Partial<Omit<SiweMessage,
          'version' | 'domain' | 'uri' | 'statement'>>
      }
    }
  | {
      chain: 'ethereum'
      message: string       // Pre-built SIWE message
      signature: Hex        // secp256k1 signature
      options?: { captchaToken?: string }
    }

type SolanaWeb3Credentials =
  | {
      chain: 'solana'
      wallet?: SolanaWallet  // defaults to window.solana
      statement?: string
      options?: {
        url?: string
        captchaToken?: string
        signInWithSolana?: Partial<Omit<SolanaSignInInput,
          'version' | 'chain' | 'domain' | 'uri' | 'statement'>>
      }
    }
  | {
      chain: 'solana'
      message: string         // Pre-built SIWS message
      signature: Uint8Array   // Ed25519 signature
      options?: { captchaToken?: string }
    }
```

**Purpose:** Sign in by verifying a message signed by user's Web3 wallet (Ethereum or Solana)

**Parameters:**

- `credentials.chain` (`'ethereum' | 'solana'`) - Blockchain to use
- `credentials.wallet?` (EthereumWallet | SolanaWallet) - Wallet interface. Defaults to `window.ethereum` / `window.solana`
- `credentials.statement?` (string) - Human-readable statement in sign-in message. No newlines. Many wallets require this
- `credentials.message?` (string) - Pre-built SIWE/SIWS message (alternative to wallet-based flow)
- `credentials.signature?` (Hex | Uint8Array) - Pre-computed signature (required with `message`)
- `credentials.options.url?` (string) - URL for message domain/URI. Defaults to `window.location.href`
- `credentials.options.captchaToken?` (string) - CAPTCHA token
- `credentials.options.signInWithEthereum?` - Override SIWE message fields (chainId, nonce, issuedAt, expirationTime, notBefore, requestId, resources)
- `credentials.options.signInWithSolana?` - Override SIWS message fields (similar to SIWE)

**Returns:**

```ts
{ data: { session: Session, user: User }, error: null }
// or
{ data: { session: null, user: null }, error: AuthError }
```

**Usage:**

```ts
// Ethereum - auto-detect wallet (browser)
const { data, error } = await client.auth.signInWithWeb3({
  chain: 'ethereum',
  statement: 'Sign in to MyApp',
})

// Ethereum - explicit wallet object
const { data, error } = await client.auth.signInWithWeb3({
  chain: 'ethereum',
  wallet: window.ethereum,
  statement: 'Sign in to MyApp',
})

// Ethereum - pre-signed message (server-side or custom flow)
const { data, error } = await client.auth.signInWithWeb3({
  chain: 'ethereum',
  message: siweMessageString,
  signature: '0xabc123...',
})

// Solana - auto-detect wallet
const { data, error } = await client.auth.signInWithWeb3({
  chain: 'solana',
  statement: 'Sign in to MyApp',
})

// Solana - explicit wallet
const { data, error } = await client.auth.signInWithWeb3({
  chain: 'solana',
  wallet: window.solana,
  statement: 'Sign in to MyApp',
})

// Non-browser environment (must provide wallet + url)
const { data, error } = await client.auth.signInWithWeb3({
  chain: 'ethereum',
  wallet: customWalletProvider,
  statement: 'Sign in to MyApp',
  options: {
    url: 'https://myapp.com',
  },
})
```

**Implementation Considerations:**

- **Backend Requirements:** Web3 token grant type endpoint, SIWE/SIWS message parsing, signature verification (secp256k1 for Ethereum, Ed25519 for Solana)
- **Security:** EIP-4361 standard prevents replay attacks via nonce/issuedAt/expirationTime; domain binding prevents phishing; wallet must be connected before signing
- **Complexity:** Complex - wallet integration, message construction (SIWE/SIWS formats), signature verification, two distinct chain implementations
- **Dependencies:** Web3 wallet (MetaMask, Phantom, etc.); `window.ethereum` or `window.solana` in browser; explicit wallet object in non-browser
- **Storage:** Session saved to storage, SIGNED_IN event emitted

**Authentication Flow:**

```
# Wallet-based flow (Ethereum example):
signInWithWeb3({ chain: 'ethereum', wallet?, statement })
  → Resolve wallet (explicit or window.ethereum)
  → eth_requestAccounts → get user address
  → eth_chainId → get chain ID
  → Construct SIWE message (domain, address, statement, uri, version, chainId, nonce, issuedAt)
  → personal_sign(message, address) → get signature
  → POST /token?grant_type=web3
      → Body: { chain: 'ethereum', message, signature, gotrue_meta_security }
  → Backend:
      → Parse SIWE/SIWS message
      → Verify signature against message
      → Extract wallet address
      → Find or create user by chain + address
      → Issue session
  → Client:
      → Save session to storage
      → Notify subscribers (SIGNED_IN)

# Pre-signed message flow:
signInWithWeb3({ chain, message, signature })
  → POST /token?grant_type=web3
      → Body: { chain, message, signature }
  → (same backend flow)
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /token?grant_type=web3`
- **Request Body (Ethereum):**

```json
{
  "chain": "ethereum",
  "message": "myapp.com wants you to sign in with your Ethereum account:\n0x1234...\n\nSign in to MyApp\n\nURI: https://myapp.com\nVersion: 1\nChain ID: 1\nNonce: abc123\nIssued At: 2024-01-01T00:00:00.000Z",
  "signature": "0xabc123...",
  "gotrue_meta_security": { "captcha_token": "..." }
}
```

- **Request Body (Solana):**

```json
{
  "chain": "solana",
  "message": "myapp.com wants you to sign in with your Solana account:\nABC123...\n\nSign in to MyApp\n\nVersion: 1\nURI: https://myapp.com\nIssued At: 2024-01-01T00:00:00.000Z",
  "signature": "base64url_encoded_ed25519_signature",
  "gotrue_meta_security": { "captcha_token": "..." }
}
```

- **Response:** Same as standard token response (access_token, refresh_token, user)

**Error Cases:**

- `No compatible Ethereum wallet interface` - `window.ethereum` not found and no wallet provided
- `No compatible Solana wallet interface` - `window.solana` not found and no wallet provided
- `No accounts available` - wallet connected but no accounts
- `Wallet method eth_requestAccounts is missing` - incompatible Ethereum wallet
- `Wallet does not have a compatible signMessage()` - incompatible Solana wallet (no `signIn` or `signMessage`)
- `Both wallet and url must be specified in non-browser environments`
- `AuthInvalidTokenResponseError` - backend returned no session/user
- `Unsupported chain` - chain not `ethereum` or `solana`

**Related:** `signInWithIdToken`, `signInWithOAuth`

---

### `signInAnonymously(credentials?)`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async signInAnonymously(
  credentials?: SignInAnonymouslyCredentials
): Promise<AuthResponse>

type SignInAnonymouslyCredentials = {
  options?: {
    data?: object
    captchaToken?: string
  }
}
```

**Purpose:** Create an anonymous user with a session (is_anonymous=true in JWT)

**Parameters:**

- `credentials.options.data?` (object) - Custom user metadata (maps to `auth.users.raw_user_meta_data`)
- `credentials.options.captchaToken?` (string) - CAPTCHA verification token

**Returns:**

```ts
{ data: { user: User, session: Session }, error: null }
// or
{ data: { user: null, session: null }, error: AuthError }
```

**Usage:**

```ts
// Basic anonymous sign-in
const { data, error } = await client.auth.signInAnonymously()
// data.session.access_token has is_anonymous: true claim
// data.user.is_anonymous === true

// With custom metadata
const { data, error } = await client.auth.signInAnonymously({
  options: {
    data: { preferred_language: 'en', theme: 'dark' },
  },
})

// With CAPTCHA
const { data, error } = await client.auth.signInAnonymously({
  options: {
    captchaToken: 'captcha_token...',
  },
})

// Convert anonymous user to permanent (later)
const { data, error } = await client.auth.updateUser({
  email: 'user@example.com',
  password: 'securePassword123!',
})
// User is no longer anonymous after linking credentials
```

**Implementation Considerations:**

- **Backend Requirements:** Signup endpoint accepting empty credentials, anonymous user flag in DB, is_anonymous JWT claim
- **Security:** Anonymous users should have limited permissions via RLS; no email/password required; CAPTCHA recommended to prevent abuse
- **Complexity:** Simple - uses same signup endpoint with no email/password
- **Dependencies:** None (no email/SMS services needed)
- **Storage:** Session saved to storage, SIGNED_IN event emitted

**Authentication Flow:**

```
signInAnonymously({ options? })
  → POST /signup
      → Body: { data: options.data ?? {}, gotrue_meta_security: { captcha_token } }
  → Backend:
      → Create user with is_anonymous=true
      → No email/password stored
      → Issue session with is_anonymous claim in JWT
  → Client:
      → Save session to storage
      → Notify subscribers (SIGNED_IN)
      → Return { data: { user, session }, error: null }
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /signup`
- **Request Body:**

```json
{
  "data": { "preferred_language": "en" },
  "gotrue_meta_security": { "captcha_token": "..." }
}
```

- **Response:**

```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "bearer",
  "expires_in": 3600,
  "expires_at": 1700000000,
  "refresh_token": "...",
  "user": {
    "id": "uuid",
    "is_anonymous": true,
    "user_metadata": { "preferred_language": "en" },
    ...
  }
}
```

**Error Cases:**

- `captcha_failed` - CAPTCHA verification failed
- `signup_disabled` - anonymous signups disabled on server
- Network/server errors

**Related:** `signUp`, `updateUser` (to convert anonymous → permanent)

---

### `startAutoRefresh()`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async startAutoRefresh(): Promise<void>
```

**Purpose:** Start background token refresh process (checks every 30 seconds)

**Parameters:** None

**Returns:**

```ts
void
```

**Usage:**

```ts
// Manual auto-refresh (React Native / non-browser)
// Start when app comes to foreground
client.auth.startAutoRefresh()

// Stop when app goes to background
client.auth.stopAutoRefresh()

// React Native example with AppState
import { AppState } from 'react-native'

AppState.addEventListener('change', (state) => {
  if (state === 'active') {
    client.auth.startAutoRefresh()
  } else {
    client.auth.stopAutoRefresh()
  }
})

// Not needed if autoRefreshToken option is true (default)
// The client manages visibility changes automatically in browsers
const client = createClient(url, key, {
  auth: { autoRefreshToken: true }, // default
})
```

**Implementation Considerations:**

- **Backend Requirements:** Token refresh endpoint (`POST /token?grant_type=refresh_token`)
- **Security:** Removes managed visibilitychange callback; caller must manage visibility changes manually after calling this
- **Complexity:** Simple - sets 30-second interval, runs tick immediately
- **Dependencies:** None (uses built-in timers)
- **Storage:** Reads session from storage to check expiry; updates session on successful refresh

**Authentication Flow:**

```
startAutoRefresh()
  → Remove any managed visibilitychange callback
  → Stop existing auto-refresh (if running)
  → Set 30-second interval → _autoRefreshTokenTick()
  → Run first tick immediately (next event loop)
  → Each tick:
      → Read session from storage
      → Check if token expires within threshold (~3 ticks = 90 seconds)
      → If expiring: POST /token?grant_type=refresh_token
      → Save new session, notify TOKEN_REFRESHED
      → If refresh fails: retry with exponential backoff
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /token?grant_type=refresh_token` (called internally by tick)
- **Request Body:**

```json
{
  "refresh_token": "..."
}
```

- **Response:** New session (access_token, refresh_token, expires_in, user)

**Error Cases:**

- None directly - errors handled internally by refresh tick with retry logic

**Related:** `stopAutoRefresh`, `refreshSession`, `onAuthStateChange` (TOKEN_REFRESHED event)

---

### `stopAutoRefresh()`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async stopAutoRefresh(): Promise<void>
```

**Purpose:** Stop background token refresh process

**Parameters:** None

**Returns:**

```ts
void
```

**Usage:**

```ts
// Stop auto-refresh when app backgrounds
client.auth.stopAutoRefresh()

// React Native lifecycle
import { AppState } from 'react-native'

AppState.addEventListener('change', (state) => {
  if (state === 'active') {
    client.auth.startAutoRefresh()
  } else {
    client.auth.stopAutoRefresh()
  }
})

// Desktop/Electron app
window.addEventListener('blur', () => {
  client.auth.stopAutoRefresh()
})
window.addEventListener('focus', () => {
  client.auth.startAutoRefresh()
})
```

**Implementation Considerations:**

- **Backend Requirements:** None
- **Security:** Removes managed visibilitychange callback; caller must manage visibility changes manually
- **Complexity:** Simple - clears interval and timeout
- **Dependencies:** None
- **Storage:** No storage changes

**Authentication Flow:**

```
stopAutoRefresh()
  → Remove any managed visibilitychange callback
  → Clear auto-refresh interval timer
  → Clear pending tick timeout
  → No more background refresh until startAutoRefresh() called
```

**GoTrue Backend Mapping:**

- No backend calls - client-side only

**Error Cases:**

- None - safe to call even if auto-refresh not running

**Related:** `startAutoRefresh`, `refreshSession`

---

## Summary

**6 advanced authentication methods documented:**

- signInWithSSO - Enterprise SSO via SAML identity provider
- signInWithIdToken - OIDC ID token exchange (Google, Apple, etc.)
- signInWithWeb3 - Wallet-based auth (Ethereum SIWE, Solana SIWS)
- signInAnonymously - Anonymous user creation with is_anonymous JWT claim
- startAutoRefresh - Start 30-second background token refresh
- stopAutoRefresh - Stop background token refresh

**Key Patterns:**

1. **Token Grant Types:** SSO uses `/sso`, IdToken uses `grant_type=id_token`, Web3 uses `grant_type=web3`, Anonymous uses `/signup`
2. **Session Handling:** signInWithSSO returns URL (no session yet); signInWithIdToken, signInWithWeb3, signInAnonymously all return session directly
3. **PKCE Support:** signInWithSSO generates code challenge automatically when flowType is `pkce`
4. **Auto-Refresh:** 30-second tick interval, refreshes ~90 seconds before expiry, exponential backoff on failure
5. **Visibility Management:** startAutoRefresh/stopAutoRefresh remove managed visibilitychange callback; manual management required after calling

**Common Use Cases:**

- **Enterprise login:** signInWithSSO({ domain: 'acme.com' })
- **Mobile native auth:** signInWithIdToken({ provider: 'google', token })
- **Web3 dApp:** signInWithWeb3({ chain: 'ethereum', statement: '...' })
- **Guest checkout:** signInAnonymously() → later updateUser({ email, password })
- **React Native lifecycle:** startAutoRefresh/stopAutoRefresh on AppState change
