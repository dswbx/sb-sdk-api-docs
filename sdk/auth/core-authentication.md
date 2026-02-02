# Core Authentication

**Priority:** HIGH
**Methods:** 13
**Status:** Complete

Core authentication flows for session management, sign in, sign up, sign out, and verification.

---

## Session Management (5 methods)

### `getSession()`

**Priority:** High | **Complexity:** Simple

**Signature:**

```ts
async getSession(): Promise<AuthResponse>

// Return type
AuthResponse = {
  data: { user: User | null; session: Session | null }
  error: AuthError | null
}
```

**Purpose:** Returns current session, auto-refreshing if expired

**Parameters:** None

**Returns:**

```ts
{ data: { user: User | null, session: Session | null }, error: AuthError | null }
```

**Usage:**

```ts
// Get current session
const { data: { session }, error } = await client.auth.getSession()

if (error) {
  console.error('Session error:', error.message)
} else if (session) {
  console.log('User:', session.user.email)
  console.log('Expires:', new Date(session.expires_at * 1000))
}
```

**Implementation Considerations:**

- **Backend Requirements:** None - reads from local storage
- **Security:** WARNING - loads from storage directly (may be request cookies). Not authenticated. Use `getUser()` for auth decisions.
- **Complexity:** Simple - no network request unless refresh needed
- **Dependencies:** Storage adapter (localStorage or custom)
- **Storage:** Reads session from `${storageKey}` key

**Session Storage Format:**

```ts
{
  access_token: string    // JWT
  refresh_token: string   // Refresh token
  expires_in: number      // Seconds until expiry
  expires_at: number      // Unix timestamp
  token_type: 'bearer'
  user: User
}
```

**Authentication Flow:**

```
getSession()
  → Acquire lock (prevent race conditions)
  → Load session from storage
  → Check expiration (current time + EXPIRY_MARGIN_MS)
  → If expired: call refreshSession()
  → Return session
```

**GoTrue Backend Mapping:**

- **Endpoint:** None (local operation)
- **Refresh Endpoint (if expired):** `POST /token?grant_type=refresh_token`

**Error Cases:**

- No errors - returns `{ session: null, user: null }` if not authenticated

**Related:** `setSession`, `refreshSession`, `getUser`

---

### `setSession(currentSession)`

**Priority:** High | **Complexity:** Moderate

**Signature:**

```ts
async setSession(currentSession: {
  access_token: string
  refresh_token: string
}): Promise<AuthResponse>
```

**Purpose:** Set session from tokens, auto-refresh if expired, validate user

**Parameters:**

- `currentSession.access_token` (string) - JWT access token
- `currentSession.refresh_token` (string) - Refresh token

**Returns:**

```ts
{ data: { user: User, session: Session }, error: AuthError | null }
```

**Usage:**

```ts
// Set session from external tokens (e.g., server-side auth)
const { data, error } = await client.auth.setSession({
  access_token: 'eyJhbGci...',
  refresh_token: 'v1.abc...',
})

if (error) {
  console.error('Invalid session:', error.message)
} else {
  console.log('Session set for:', data.user.email)
}

// Restore session after page reload
const savedSession = JSON.parse(localStorage.getItem('my-session'))
await client.auth.setSession(savedSession)
```

**Implementation Considerations:**

- **Backend Requirements:** JWT validation, user lookup endpoint, token refresh endpoint
- **Security:** Decodes JWT locally, validates signature via `getUser()` network call
- **Complexity:** Moderate - JWT decode + network validation + possible refresh
- **Dependencies:** JWT library for decode
- **Storage:** Saves validated session to storage, notifies subscribers

**Authentication Flow:**

```
setSession({ access_token, refresh_token })
  → Decode JWT to extract payload (exp, sub, etc.)
  → Check expiration
  → If expired:
      → POST /token?grant_type=refresh_token
      → Get new access_token and refresh_token
  → If valid or refreshed:
      → GET /user (with JWT) to validate authenticity
      → Save session to storage
      → Notify subscribers (SIGNED_IN event)
  → Return { user, session }
```

**GoTrue Backend Mapping:**

- **Validation Endpoint:** `GET /user`
  - **Headers:** `Authorization: Bearer <access_token>`
  - **Response:** `{ id, email, user_metadata, ... }`
- **Refresh Endpoint (if expired):** `POST /token?grant_type=refresh_token`
  - **Body:** `{ refresh_token }`
  - **Response:** `{ access_token, refresh_token, expires_in, user }`

**Error Cases:**

- `AuthSessionMissingError` - tokens missing
- `invalid_grant` - refresh token invalid/expired
- `invalid_jwt` - access token malformed
- Network errors from user fetch

**Related:** `getSession`, `refreshSession`, `exchangeCodeForSession`

---

### `refreshSession(currentSession?)`

**Priority:** High | **Complexity:** Simple

**Signature:**

```ts
async refreshSession(currentSession?: {
  refresh_token: string
}): Promise<AuthResponse>
```

**Purpose:** Force refresh session regardless of expiry

**Parameters:**

- `currentSession?` (optional) - Object with refresh_token. If omitted, retrieves from storage.

**Returns:**

```ts
{ data: { user: User, session: Session }, error: AuthError | null }
```

**Usage:**

```ts
// Force refresh current session
const { data, error } = await client.auth.refreshSession()

if (error) {
  console.error('Refresh failed:', error.message)
  // Redirect to login
} else {
  console.log('New access token:', data.session.access_token)
}

// Refresh with explicit refresh token
const { data, error } = await client.auth.refreshSession({
  refresh_token: 'v1.stored_token...',
})
```

**Implementation Considerations:**

- **Backend Requirements:** Token refresh endpoint, validate refresh token, generate new JWT
- **Security:** Refresh tokens are single-use (rotation), old token invalidated
- **Complexity:** Simple - single API call
- **Dependencies:** Token storage
- **Storage:** Saves new session, replaces old tokens

**Authentication Flow:**

```
refreshSession(currentSession?)
  → If no currentSession: load from storage
  → Extract refresh_token
  → POST /token?grant_type=refresh_token
      → Body: { refresh_token }
  → Backend: Validate refresh token, generate new JWT + refresh token
  → Save new session to storage
  → Return { user, session }
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /token?grant_type=refresh_token`
- **Request Body:** `{ refresh_token: string }`
- **Response:**

```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "v1.new_token...",
  "expires_in": 3600,
  "token_type": "bearer",
  "user": { "id": "...", "email": "...", ... }
}
```

**Error Cases:**

- `AuthSessionMissingError` - no refresh token
- `invalid_grant` - refresh token invalid, expired, or revoked
- `invalid_request` - malformed request

**Related:** `getSession`, `setSession`

---

### `getUser(jwt?)`

**Priority:** High | **Complexity:** Simple

**Signature:**

```ts
async getUser(jwt?: string): Promise<UserResponse>

// Return type
UserResponse = {
  data: { user: User }
  error: AuthError | null
}
```

**Purpose:** Get authenticated user details via network request (authentic, use for authorization)

**Parameters:**

- `jwt?` (string, optional) - Access token JWT. If omitted, uses current session token.

**Returns:**

```ts
{ data: { user: User }, error: AuthError | null }
```

**Usage:**

```ts
// Get current user (makes network request)
const { data: { user }, error } = await client.auth.getUser()

if (error) {
  console.error('Not authenticated:', error.message)
} else {
  console.log('User ID:', user.id)
  console.log('Email:', user.email)
  console.log('Metadata:', user.user_metadata)
}

// Validate specific JWT
const { data, error } = await client.auth.getUser('eyJhbGci...')
```

**Implementation Considerations:**

- **Backend Requirements:** User lookup endpoint, JWT validation
- **Security:** ALWAYS makes network request - authentic data, safe for authorization decisions
- **Complexity:** Simple - single GET request
- **Dependencies:** Active session or JWT
- **Storage:** Does not modify storage

**Authentication Flow:**

```
getUser(jwt?)
  → If jwt provided: use jwt
  → Else: load JWT from current session
  → GET /user with Authorization header
  → Backend: Validate JWT signature, check expiration, lookup user
  → Return user object
```

**GoTrue Backend Mapping:**

- **Endpoint:** `GET /user`
- **Headers:** `Authorization: Bearer <jwt>`
- **Response:**

```json
{
  "id": "uuid",
  "email": "user@example.com",
  "phone": "+1234567890",
  "email_confirmed_at": "2024-01-01T00:00:00Z",
  "phone_confirmed_at": null,
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z",
  "user_metadata": {},
  "app_metadata": { "provider": "email" },
  "identities": [...]
}
```

**Error Cases:**

- `AuthSessionMissingError` - no JWT or session
- `invalid_jwt` - JWT expired or malformed
- `401 Unauthorized` - invalid signature
- Network errors

**Related:** `getSession`, `updateUser`

---

### `initialize()`

**Priority:** High | **Complexity:** Moderate

**Signature:**

```ts
async initialize(): Promise<InitializeResult>

// Return type
InitializeResult = { error: AuthError | null }
```

**Purpose:** Initialize client session from URL (OAuth/magic link callback) or storage

**Parameters:** None

**Returns:**

```ts
{ error: AuthError | null }
```

**Usage:**

```ts
// Called automatically in constructor
// Manual call after redirect to check for errors
const { error } = await client.auth.initialize()

if (error) {
  if (error.message.includes('identity_already_exists')) {
    console.log('This OAuth account is already linked to another user')
  } else {
    console.error('Auth error:', error.message)
  }
}

// Detect callback type
const url = window.location.href
if (url.includes('access_token=') || url.includes('code=')) {
  // OAuth callback detected - initialize() handles it
}
```

**Implementation Considerations:**

- **Backend Requirements:** Token endpoint (for PKCE), session validation
- **Security:** Detects URL parameters (callback detection), validates PKCE code verifier
- **Complexity:** Moderate - URL parsing, PKCE exchange, storage recovery, event coordination
- **Dependencies:** Browser environment for URL detection, storage for code verifier
- **Storage:** Saves session from callback or loads from storage

**Authentication Flow:**

```
initialize()
  → Detect callback type from URL:
      - Implicit: ?access_token=... → extract tokens from fragment
      - PKCE: ?code=... → exchangeCodeForSession(code)
      - None: load session from storage → _recoverAndRefresh()
  → If callback detected:
      → Parse URL parameters
      → Extract session/code
      → Save session to storage (implicit) or exchange code (PKCE)
      → Notify subscribers (SIGNED_IN or PASSWORD_RECOVERY)
      → Clean URL (remove sensitive params)
  → If no callback:
      → Load session from storage
      → Refresh if expired
  → Return { error }
```

**GoTrue Backend Mapping:**

- **PKCE Exchange:** `POST /token?grant_type=pkce`
  - **Body:** `{ auth_code, code_verifier }`
  - **Response:** `{ access_token, refresh_token, user, ... }`

**Callback URL Formats:**

```
# Implicit flow (legacy)
https://app.com/auth/callback#access_token=xxx&refresh_token=yyy&expires_in=3600&type=signup

# PKCE flow (recommended)
https://app.com/auth/callback?code=abc123

# Magic link
https://app.com/auth/callback?token_hash=def456&type=magiclink

# Password recovery
https://app.com/auth/callback?token_hash=ghi789&type=recovery

# Error
https://app.com/auth/callback?error=access_denied&error_description=...
```

**Error Cases:**

- `AuthImplicitGrantRedirectError` - URL parsing errors, invalid callback
- `identity_already_exists` - OAuth account already linked to different user
- `identity_not_found` - identity linking error
- `single_identity_not_deletable` - cannot remove last identity
- `AuthUnknownError` - unexpected errors
- Never throws - returns error in result

**Related:** `exchangeCodeForSession`, `signInWithOAuth`, `verifyOtp`

---

## Sign In Methods (4 methods)

### `signInWithPassword(credentials)`

**Priority:** High | **Complexity:** Simple

**Signature:**

```ts
async signInWithPassword(
  credentials: SignInWithPasswordCredentials
): Promise<AuthTokenResponsePassword>

// Credentials type
SignInWithPasswordCredentials =
  | { email: string; password: string; options?: { captchaToken?: string } }
  | { phone: string; password: string; options?: { captchaToken?: string } }
```

**Purpose:** Sign in existing user with email/phone + password

**Parameters:**

- `credentials.email` (string) - User email (either email or phone required)
- `credentials.phone` (string) - User phone (either email or phone required)
- `credentials.password` (string) - User password
- `credentials.options.captchaToken?` (string) - Optional CAPTCHA token for bot protection

**Returns:**

```ts
{
  data: {
    user: User
    session: Session
    weakPassword?: { reasons: string[] }  // If password is weak
  }
  error: AuthError | null
}
```

**Usage:**

```ts
// Email + password
const { data, error } = await client.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'password123',
})

if (error) {
  console.error('Sign in failed:', error.message)
} else {
  console.log('Signed in:', data.user.email)
  if (data.weakPassword) {
    console.warn('Weak password:', data.weakPassword.reasons)
  }
}

// Phone + password
const { data, error } = await client.auth.signInWithPassword({
  phone: '+12025551234',
  password: 'password123',
})

// With CAPTCHA
const { data, error } = await client.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'password123',
  options: { captchaToken: 'recaptcha_token...' },
})
```

**Implementation Considerations:**

- **Backend Requirements:** Password hashing (bcrypt), JWT generation, rate limiting, user lookup
- **Security:** Passwords hashed with bcrypt, CAPTCHA support, error messages don't distinguish account existence
- **Complexity:** Simple - password verify + JWT generation
- **Dependencies:** Bcrypt library, JWT library, optional CAPTCHA verification
- **Storage:** Saves session on success, notifies SIGNED_IN event

**Authentication Flow:**

```
signInWithPassword({ email, password })
  → POST /token?grant_type=password
  → Backend:
      → Lookup user by email/phone
      → Verify password hash (bcrypt.compare)
      → If valid: generate JWT + refresh token
      → Check password strength (optional)
      → Return tokens + user
  → Client: Save session to storage
  → Notify subscribers (SIGNED_IN)
  → Return { user, session, weakPassword? }
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /token?grant_type=password`
- **Request Body:**

```json
{
  "email": "user@example.com",
  "password": "password123",
  "gotrue_meta_security": {
    "captcha_token": "..."
  }
}
```

- **Response:**

```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "v1.abc...",
  "expires_in": 3600,
  "token_type": "bearer",
  "user": { "id": "...", "email": "...", ... },
  "weak_password": {
    "reasons": ["length", "character_types"]
  }
}
```

**Error Cases:**

- `invalid_credentials` - wrong email/phone or password, or account doesn't exist, or social-only account
- `email_not_confirmed` - email not verified (if confirmation required)
- `phone_not_confirmed` - phone not verified
- `over_request_rate_limit` - too many attempts
- `captcha_verification_failed` - invalid CAPTCHA

**Related:** `signUp`, `signInWithOtp`, `resetPasswordForEmail`

---

### `signInWithOAuth(credentials)`

**Priority:** High | **Complexity:** Moderate

**Signature:**

```ts
async signInWithOAuth(
  credentials: SignInWithOAuthCredentials
): Promise<OAuthResponse>

// Credentials type
SignInWithOAuthCredentials = {
  provider: Provider  // 'google', 'github', 'apple', etc.
  options?: {
    redirectTo?: string
    scopes?: string
    queryParams?: { [key: string]: string }
    skipBrowserRedirect?: boolean
  }
}
```

**Purpose:** Sign in via OAuth provider (Google, GitHub, Apple, etc.) with PKCE support

**Parameters:**

- `credentials.provider` (Provider) - OAuth provider: 'google' | 'github' | 'apple' | 'azure' | etc.
- `credentials.options.redirectTo?` (string) - URL to redirect after OAuth (defaults to current URL)
- `credentials.options.scopes?` (string) - OAuth scopes (space-separated, e.g., 'email profile')
- `credentials.options.queryParams?` (object) - Additional query params for provider
- `credentials.options.skipBrowserRedirect?` (boolean) - If true, returns URL without redirecting

**Returns:**

```ts
{
  data: { provider: Provider, url: string }
  error: AuthError | null
}
```

**Usage:**

```ts
// Basic OAuth sign in (auto-redirects)
const { data, error } = await client.auth.signInWithOAuth({
  provider: 'google',
})
// Browser redirects to Google OAuth consent screen
// After consent, redirects back to app
// initialize() detects callback and creates session

// Custom redirect URL
const { data, error } = await client.auth.signInWithOAuth({
  provider: 'github',
  options: {
    redirectTo: 'https://myapp.com/auth/callback',
  },
})

// Custom scopes (GitHub example)
const { data, error } = await client.auth.signInWithOAuth({
  provider: 'github',
  options: {
    scopes: 'user:email repo',
  },
})

// Skip auto-redirect (manual control)
const { data, error } = await client.auth.signInWithOAuth({
  provider: 'google',
  options: {
    skipBrowserRedirect: true,
  },
})
if (data?.url) {
  // Manually redirect or open popup
  window.location.href = data.url
}
```

**Implementation Considerations:**

- **Backend Requirements:** OAuth provider registration (client ID/secret), provider SDKs, callback endpoint
- **Security:** PKCE flow (code verifier/challenge), state parameter for CSRF protection
- **Complexity:** Moderate - OAuth handshake, provider API integration, account linking
- **Dependencies:** Provider OAuth apps configured in GoTrue, PKCE code verifier storage
- **Storage:** Stores code verifier (for PKCE), clears after exchange

**Authentication Flow:**

```
signInWithOAuth({ provider: 'google' })
  → Generate PKCE code_verifier + code_challenge (if flowType='pkce')
  → Store code_verifier in storage
  → Build authorization URL:
      → GET /authorize?provider=google&redirect_to=...&code_challenge=...
  → Browser redirects to Google OAuth consent
  → User approves
  → Google redirects back: ?code=abc123
  → initialize() detects callback
  → exchangeCodeForSession(code)
      → POST /token?grant_type=pkce { auth_code, code_verifier }
      → Backend: Exchange code with Google, create/link user
  → Session created
```

**GoTrue Backend Mapping:**

- **Authorization URL:** `GET /authorize`
  - **Query Params:**
    - `provider` - OAuth provider name
    - `redirect_to` - Callback URL
    - `code_challenge` - PKCE challenge (if enabled)
    - `code_challenge_method` - 'S256'
    - `scopes` - OAuth scopes
- **Backend Flow:**
  1. Redirect user to provider OAuth consent
  2. Provider redirects back with code
  3. Exchange code for provider tokens
  4. Fetch user info from provider
  5. Create user or link identity
  6. Return auth code (PKCE) or session (implicit)

**PKCE vs Implicit Flow:**

| Flow | Callback | Security |
|------|----------|----------|
| **PKCE (recommended)** | `?code=abc` | More secure - tokens not in URL |
| **Implicit (legacy)** | `#access_token=xyz` | Less secure - tokens in URL fragment |

**Error Cases:**

- `access_denied` - user cancelled OAuth
- `server_error` - provider error
- `temporarily_unavailable` - provider down
- `provider_not_configured` - provider not enabled in GoTrue

**Related:** `exchangeCodeForSession`, `initialize`, `linkIdentity`

---

### `signInWithOtp(credentials)`

**Priority:** High | **Complexity:** Moderate

**Signature:**

```ts
async signInWithOtp(
  credentials: SignInWithPasswordlessCredentials
): Promise<AuthOtpResponse>

// Credentials type
SignInWithPasswordlessCredentials =
  | {
      email: string
      options?: {
        emailRedirectTo?: string
        shouldCreateUser?: boolean
        data?: object
        captchaToken?: string
      }
    }
  | {
      phone: string
      options?: {
        shouldCreateUser?: boolean
        data?: object
        captchaToken?: string
        channel?: 'sms' | 'whatsapp'
      }
    }
```

**Purpose:** Passwordless sign in via magic link (email) or OTP (email/SMS)

**Parameters:**

- `credentials.email` (string) - User email (for email OTP/magic link)
- `credentials.phone` (string) - User phone (for SMS OTP)
- `credentials.options.emailRedirectTo?` (string) - URL to redirect after clicking magic link
- `credentials.options.shouldCreateUser?` (boolean) - Create user if doesn't exist (default: true)
- `credentials.options.data?` (object) - User metadata for new users
- `credentials.options.captchaToken?` (string) - CAPTCHA token
- `credentials.options.channel?` ('sms' | 'whatsapp') - SMS delivery channel (phone only)

**Returns:**

```ts
{
  data: {
    user: null         // No session yet - must verify OTP
    session: null
    messageId?: string // SMS message ID (phone only)
  }
  error: AuthError | null
}
```

**Usage:**

```ts
// Email magic link
const { data, error } = await client.auth.signInWithOtp({
  email: 'user@example.com',
  options: {
    emailRedirectTo: 'https://myapp.com/welcome',
  },
})
if (!error) {
  console.log('Check your email for magic link')
}

// Email OTP (6-digit code)
const { data, error } = await client.auth.signInWithOtp({
  email: 'user@example.com',
})
// User receives: "Your code is 123456"
// Then call verifyOtp({ email, token: '123456', type: 'email' })

// Phone SMS OTP
const { data, error } = await client.auth.signInWithOtp({
  phone: '+12025551234',
  options: {
    channel: 'sms',
  },
})
console.log('SMS sent, messageId:', data?.messageId)
// Then call verifyOtp({ phone, token: '123456', type: 'sms' })

// Phone WhatsApp OTP
const { data, error } = await client.auth.signInWithOtp({
  phone: '+12025551234',
  options: {
    channel: 'whatsapp',
  },
})

// Prevent account creation (existing users only)
const { data, error } = await client.auth.signInWithOtp({
  email: 'user@example.com',
  options: {
    shouldCreateUser: false,
  },
})

// With user metadata (for new users)
const { data, error } = await client.auth.signInWithOtp({
  email: 'user@example.com',
  options: {
    data: {
      name: 'John Doe',
      company: 'Acme Inc',
    },
  },
})
```

**Implementation Considerations:**

- **Backend Requirements:** Email service (SMTP, SendGrid, etc.), SMS provider (Twilio), OTP generation, token storage
- **Security:** OTP expires (typically 1 hour), single-use, PKCE support for email
- **Complexity:** Moderate - email/SMS delivery, template rendering, token generation
- **Dependencies:** Email provider, SMS provider (Twilio for WhatsApp)
- **Storage:** Stores PKCE code verifier (email only), stores OTP hash

**Authentication Flow:**

```
# Magic Link Flow (Email)
signInWithOtp({ email })
  → POST /otp { email, code_challenge }
  → Backend:
      → Generate token_hash
      → Render email template with magic link
      → {{ .ConfirmationURL }}?token_hash=...&type=magiclink
      → Send email
  → User clicks link
  → initialize() detects ?token_hash=...
  → verifyOtp({ token_hash, type: 'magiclink' })
  → Session created

# OTP Flow (Email/Phone)
signInWithOtp({ email or phone })
  → POST /otp { email/phone }
  → Backend:
      → Generate 6-digit OTP
      → Store OTP hash
      → Render email/SMS template with {{ .Token }}
      → Send email/SMS
  → User receives code
  → verifyOtp({ email/phone, token: '123456', type: 'email'/'sms' })
  → Session created
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /otp`
- **Request Body (Email):**

```json
{
  "email": "user@example.com",
  "data": { "name": "John" },
  "create_user": true,
  "gotrue_meta_security": { "captcha_token": "..." },
  "code_challenge": "...",
  "code_challenge_method": "S256"
}
```

- **Request Body (Phone):**

```json
{
  "phone": "+12025551234",
  "channel": "sms",
  "create_user": true,
  "gotrue_meta_security": { "captcha_token": "..." }
}
```

- **Response:**

```json
{
  "message_id": "SM1234..." // Phone only
}
```

**Email Template Variables:**

- `{{ .Token }}` - 6-digit OTP (if included)
- `{{ .ConfirmationURL }}` - Magic link URL (if included)
- Include either Token or ConfirmationURL, not both

**Error Cases:**

- `over_email_send_rate_limit` - too many emails
- `over_sms_send_rate_limit` - too many SMS
- `invalid_email` - malformed email
- `invalid_phone` - malformed phone
- `user_not_found` - user doesn't exist (if `shouldCreateUser: false`)

**Related:** `verifyOtp`, `resend`

---

### `exchangeCodeForSession(authCode)`

**Priority:** High | **Complexity:** Moderate

**Signature:**

```ts
async exchangeCodeForSession(authCode: string): Promise<AuthTokenResponse>
```

**Purpose:** Exchange PKCE authorization code for session (completes OAuth/OTP flow)

**Parameters:**

- `authCode` (string) - Authorization code from OAuth callback or OTP magic link

**Returns:**

```ts
{
  data: {
    user: User
    session: Session
  }
  error: AuthError | null
}
```

**Usage:**

```ts
// Automatically called by initialize() after OAuth redirect
// Manual call if handling callback yourself:

const url = new URL(window.location.href)
const code = url.searchParams.get('code')

if (code) {
  const { data, error } = await client.auth.exchangeCodeForSession(code)

  if (error) {
    console.error('Code exchange failed:', error.message)
  } else {
    console.log('Signed in:', data.user.email)
    // Clean URL
    window.history.replaceState({}, '', url.pathname)
  }
}
```

**Implementation Considerations:**

- **Backend Requirements:** Code storage, code verifier validation (PKCE), token generation
- **Security:** PKCE code verifier required (stored during OAuth initiation), single-use codes
- **Complexity:** Moderate - PKCE validation, provider token exchange (OAuth), session creation
- **Dependencies:** Storage for code verifier
- **Storage:** Saves session, removes code verifier

**Authentication Flow:**

```
exchangeCodeForSession(authCode)
  → Load code_verifier from storage (stored during signInWithOAuth/signInWithOtp)
  → POST /token?grant_type=pkce
      → Body: { auth_code, code_verifier }
  → Backend:
      → Validate auth_code exists and not expired
      → Verify code_verifier matches code_challenge (SHA-256)
      → For OAuth: Exchange code with provider, fetch user info
      → For OTP: Validate token_hash
      → Create/link user
      → Generate JWT + refresh token
  → Save session to storage
  → Remove code_verifier from storage
  → Notify subscribers (SIGNED_IN)
  → Return { user, session }
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /token?grant_type=pkce`
- **Request Body:**

```json
{
  "auth_code": "abc123...",
  "code_verifier": "random_string_stored_during_oauth"
}
```

- **Response:**

```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "v1.abc...",
  "expires_in": 3600,
  "token_type": "bearer",
  "user": { "id": "...", "email": "...", ... }
}
```

**PKCE Security:**

```
# During OAuth initiation (signInWithOAuth)
code_verifier = random_string(43-128 chars)
code_challenge = base64url(sha256(code_verifier))
Store code_verifier in storage
Send code_challenge to authorization endpoint

# During code exchange
Load code_verifier from storage
Send code_verifier with auth_code
Backend: Verify sha256(code_verifier) == stored code_challenge
```

**Error Cases:**

- `AuthPKCECodeVerifierMissingError` - code verifier not found in storage
- `invalid_grant` - auth code invalid, expired, or already used
- `invalid_request` - code verifier doesn't match challenge
- `AuthInvalidTokenResponseError` - incomplete response

**Related:** `signInWithOAuth`, `signInWithOtp`, `initialize`

---

## Sign Up (1 method)

### `signUp(credentials)`

**Priority:** High | **Complexity:** Simple

**Signature:**

```ts
async signUp(
  credentials: SignUpWithPasswordCredentials
): Promise<AuthResponse>

// Credentials type
SignUpWithPasswordCredentials =
  | {
      email: string
      password: string
      options?: {
        emailRedirectTo?: string
        data?: object
        captchaToken?: string
      }
    }
  | {
      phone: string
      password: string
      options?: {
        data?: object
        captchaToken?: string
        channel?: 'sms' | 'whatsapp'
      }
    }
```

**Purpose:** Create new user account with email/phone + password

**Parameters:**

- `credentials.email` (string) - User email (either email or phone required)
- `credentials.phone` (string) - User phone (either email or phone required)
- `credentials.password` (string) - User password
- `credentials.options.emailRedirectTo?` (string) - URL to redirect after email confirmation
- `credentials.options.data?` (object) - User metadata (stored in `user_metadata`)
- `credentials.options.captchaToken?` (string) - CAPTCHA token
- `credentials.options.channel?` ('sms' | 'whatsapp') - SMS delivery channel (phone only)

**Returns:**

```ts
{
  data: {
    user: User
    session: Session | null  // null if email confirmation required
  }
  error: AuthError | null
}
```

**Usage:**

```ts
// Email sign up (with confirmation)
const { data, error } = await client.auth.signUp({
  email: 'user@example.com',
  password: 'securePassword123!',
})

if (error) {
  console.error('Sign up failed:', error.message)
} else if (data.session) {
  console.log('Signed up and logged in:', data.user.email)
} else {
  console.log('Check email for confirmation link')
}

// With user metadata
const { data, error } = await client.auth.signUp({
  email: 'user@example.com',
  password: 'securePassword123!',
  options: {
    data: {
      name: 'John Doe',
      age: 30,
      preferences: { theme: 'dark' },
    },
  },
})
console.log('User metadata:', data.user?.user_metadata)

// With redirect URL (after confirmation)
const { data, error } = await client.auth.signUp({
  email: 'user@example.com',
  password: 'securePassword123!',
  options: {
    emailRedirectTo: 'https://myapp.com/welcome',
  },
})

// Phone sign up
const { data, error } = await client.auth.signUp({
  phone: '+12025551234',
  password: 'securePassword123!',
  options: {
    channel: 'sms',
  },
})
// User receives SMS with OTP to confirm phone

// With CAPTCHA
const { data, error } = await client.auth.signUp({
  email: 'user@example.com',
  password: 'securePassword123!',
  options: {
    captchaToken: 'recaptcha_token...',
  },
})
```

**Implementation Considerations:**

- **Backend Requirements:** Password hashing (bcrypt), user table, email/SMS delivery, JWT generation
- **Security:** Password strength validation, bcrypt hashing, email/phone confirmation (optional), CAPTCHA support
- **Complexity:** Simple - user creation + optional confirmation
- **Dependencies:** Email service (if confirmation enabled), SMS provider (phone signup)
- **Storage:** Saves session if autoconfirm enabled, otherwise session null until confirmed

**Authentication Flow:**

```
# Email Sign Up (with confirmation)
signUp({ email, password })
  → POST /signup { email, password, data }
  → Backend:
      → Validate email format, password strength
      → Hash password (bcrypt)
      → Create user in database (status: unconfirmed)
      → Generate confirmation token
      → Send confirmation email
      → If autoconfirm enabled: generate JWT + return session
      → Else: return user only (session: null)
  → Client: Save session if returned
  → User clicks confirmation link
  → verifyOtp({ token_hash, type: 'signup' })
  → Session created

# Phone Sign Up
signUp({ phone, password })
  → POST /signup { phone, password, channel }
  → Backend:
      → Validate phone format
      → Hash password
      → Create user (status: unconfirmed)
      → Send SMS with OTP
  → User receives OTP
  → verifyOtp({ phone, token: '123456', type: 'sms' })
  → Session created
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /signup`
- **Request Body (Email):**

```json
{
  "email": "user@example.com",
  "password": "securePassword123!",
  "data": {
    "name": "John Doe",
    "age": 30
  },
  "gotrue_meta_security": {
    "captcha_token": "..."
  },
  "code_challenge": "...",
  "code_challenge_method": "S256"
}
```

- **Response (Autoconfirm ON):**

```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "v1.abc...",
  "expires_in": 3600,
  "token_type": "bearer",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "email_confirmed_at": "2024-01-01T00:00:00Z",
    "user_metadata": { "name": "John Doe", "age": 30 },
    ...
  }
}
```

- **Response (Autoconfirm OFF):**

```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "email_confirmed_at": null,  // Not confirmed yet
    "user_metadata": { "name": "John Doe", "age": 30 },
    ...
  }
}
```

**Autoconfirm Behavior:**

| Setting | Email Sent? | Session Returned? | User State |
|---------|-------------|-------------------|------------|
| **ON** | No | Yes | email_confirmed_at set |
| **OFF** | Yes | No | email_confirmed_at null |

**Error Cases:**

- `user_already_exists` - email/phone already registered (message may be vague for security)
- `invalid_email` - malformed email
- `invalid_phone` - malformed phone
- `weak_password` - password doesn't meet strength requirements
- `over_request_rate_limit` - too many signups
- `captcha_verification_failed` - invalid CAPTCHA

**Related:** `signInWithPassword`, `verifyOtp`, `resend`

---

## Sign Out (1 method)

### `signOut(options?)`

**Priority:** High | **Complexity:** Simple

**Signature:**

```ts
async signOut(options: SignOut = { scope: 'global' }): Promise<{
  error: AuthError | null
}>

// Options type
SignOut = {
  scope?: 'global' | 'local' | 'others'
}
```

**Purpose:** Sign out user, revoke tokens, clear session

**Parameters:**

- `options.scope?` ('global' | 'local' | 'others') - Scope of sign out:
  - **'global'** (default) - Sign out all sessions for this user (all devices)
  - **'local'** - Sign out only current session (this device)
  - **'others'** - Sign out all other sessions except current (other devices)

**Returns:**

```ts
{ error: AuthError | null }
```

**Usage:**

```ts
// Sign out all sessions (default)
const { error } = await client.auth.signOut()
if (!error) {
  console.log('Signed out from all devices')
  // Redirect to login page
}

// Sign out only current session
const { error } = await client.auth.signOut({ scope: 'local' })
if (!error) {
  console.log('Signed out from this device')
}

// Sign out all other sessions (keep current)
const { error } = await client.auth.signOut({ scope: 'others' })
if (!error) {
  console.log('Signed out from all other devices')
  // User stays logged in on current device
}
```

**Implementation Considerations:**

- **Backend Requirements:** Token revocation endpoint, refresh token storage
- **Security:** Revokes refresh tokens (access tokens cannot be revoked until expiry)
- **Complexity:** Simple - API call + storage cleanup
- **Dependencies:** Admin API for token revocation
- **Storage:** Clears session from local storage (except 'others' scope), removes code verifier, triggers SIGNED_OUT event

**Authentication Flow:**

```
signOut({ scope })
  → Get current session JWT
  → POST /logout (via admin.signOut)
      → Body: { scope }
      → Headers: Authorization: Bearer <jwt>
  → Backend:
      → Validate JWT
      → If scope='global': Revoke all refresh tokens for user
      → If scope='local': Revoke only this refresh token
      → If scope='others': Revoke all except this refresh token
  → Client:
      → If scope != 'others': Remove session from storage
      → Remove code verifier
      → Notify subscribers (SIGNED_OUT) if local/global
  → Return { error }
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /logout`
- **Headers:** `Authorization: Bearer <jwt>`
- **Request Body:**

```json
{
  "scope": "global"
}
```

- **Response:** 204 No Content (success)

**Scope Behavior Details:**

| Scope | Tokens Revoked | Local Storage | SIGNED_OUT Event | Use Case |
|-------|----------------|---------------|------------------|----------|
| **global** | All user tokens | Cleared | Yes | Full logout (all devices) |
| **local** | Current token | Cleared | Yes | Logout from this device only |
| **others** | All except current | NOT cleared | No | Logout other devices, stay logged in here |

**Access Token vs Refresh Token:**

- **Access Token (JWT):** Cannot be revoked - valid until expiry (typically 1 hour). Backend must validate expiration and check if associated refresh token was revoked.
- **Refresh Token:** Revoked immediately - cannot obtain new access tokens after revocation.

**Error Handling:**

- Ignores `AuthSessionMissingError` (already signed out)
- Ignores 401/403/404 errors (user may not exist or already logged out)
- Returns error for other failures (network errors, server errors)
- Never throws - always returns error in result

**Error Cases:**

- None - errors silently ignored (user is effectively signed out)

**Related:** `signInWithPassword`, `admin.signOut`

---

## Verification (2 methods)

### `verifyOtp(params)`

**Priority:** High | **Complexity:** Simple

**Signature:**

```ts
async verifyOtp(params: VerifyOtpParams): Promise<AuthResponse>

// Params types
VerifyOtpParams =
  | VerifyMobileOtpParams
  | VerifyEmailOtpParams
  | VerifyTokenHashParams

VerifyMobileOtpParams = {
  phone: string
  token: string
  type: 'sms' | 'phone_change'
  options?: { redirectTo?: string; captchaToken?: string }
}

VerifyEmailOtpParams = {
  email: string
  token: string
  type: 'signup' | 'invite' | 'magiclink' | 'recovery' | 'email_change' | 'email'
  options?: { redirectTo?: string; captchaToken?: string }
}

VerifyTokenHashParams = {
  token_hash: string
  type: EmailOtpType
}
```

**Purpose:** Verify OTP/token from email/SMS, create session

**Parameters:**

- **Email OTP:**
  - `params.email` (string) - User email
  - `params.token` (string) - 6-digit OTP code
  - `params.type` ('signup' | 'invite' | 'magiclink' | 'recovery' | 'email_change' | 'email')
- **Phone OTP:**
  - `params.phone` (string) - User phone
  - `params.token` (string) - 6-digit OTP code
  - `params.type` ('sms' | 'phone_change')
- **Token Hash (from URL):**
  - `params.token_hash` (string) - Hashed token from URL
  - `params.type` (EmailOtpType)
- **Options:**
  - `params.options.redirectTo?` (string) - URL to redirect after verification
  - `params.options.captchaToken?` (string) - CAPTCHA token

**Returns:**

```ts
{
  data: {
    user: User
    session: Session
  }
  error: AuthError | null
}
```

**Usage:**

```ts
// Verify email OTP (after signInWithOtp)
const { data, error } = await client.auth.verifyOtp({
  email: 'user@example.com',
  token: '123456',
  type: 'email',
})

// Verify SMS OTP
const { data, error } = await client.auth.verifyOtp({
  phone: '+12025551234',
  token: '654321',
  type: 'sms',
})

// Verify email confirmation (signup)
const { data, error } = await client.auth.verifyOtp({
  email: 'user@example.com',
  token: '123456',
  type: 'signup',
})

// Verify password recovery token
const { data, error } = await client.auth.verifyOtp({
  email: 'user@example.com',
  token: '789012',
  type: 'recovery',
})
// After verification, call updateUser({ password: 'newPassword' })

// Verify token hash from URL (magic link)
const url = new URL(window.location.href)
const tokenHash = url.searchParams.get('token_hash')
const type = url.searchParams.get('type')

if (tokenHash && type) {
  const { data, error } = await client.auth.verifyOtp({
    token_hash: tokenHash,
    type: type as EmailOtpType,
  })
}

// Verify email change
const { data, error } = await client.auth.verifyOtp({
  email: 'newemail@example.com',
  token: '456789',
  type: 'email_change',
})
```

**Implementation Considerations:**

- **Backend Requirements:** OTP storage, hash validation, session creation, rate limiting
- **Security:** OTP single-use, expires (typically 1 hour), rate limited
- **Complexity:** Simple - token validation + session creation
- **Dependencies:** OTP storage (database or cache)
- **Storage:** Saves session, notifies subscribers (SIGNED_IN or PASSWORD_RECOVERY)

**Authentication Flow:**

```
verifyOtp({ email, token, type })
  → POST /verify
      → Body: { email/phone, token, type, redirect_to, captcha_token }
  → Backend:
      → Lookup OTP hash by email/phone + type
      → Verify token matches hash
      → Check expiration (typically 1 hour)
      → Mark token as used (single-use)
      → Generate JWT + refresh token
      → Return session
  → Client: Save session to storage
  → Notify subscribers (SIGNED_IN or PASSWORD_RECOVERY for recovery type)
  → Return { user, session }
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /verify`
- **Request Body:**

```json
{
  "email": "user@example.com",
  "token": "123456",
  "type": "email",
  "redirect_to": "https://myapp.com/welcome",
  "gotrue_meta_security": {
    "captcha_token": "..."
  }
}
```

- **Response:**

```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "v1.abc...",
  "expires_in": 3600,
  "token_type": "bearer",
  "user": { "id": "...", "email": "...", ... }
}
```

**OTP Types:**

| Type | Use Case | Event |
|------|----------|-------|
| `email` | Email OTP sign in | SIGNED_IN |
| `sms` | SMS OTP sign in | SIGNED_IN |
| `signup` | Email confirmation after signup | SIGNED_IN |
| `invite` | Accept email invite | SIGNED_IN |
| `magiclink` | Magic link sign in | SIGNED_IN |
| `recovery` | Password reset verification | PASSWORD_RECOVERY |
| `email_change` | Confirm new email | SIGNED_IN |
| `phone_change` | Confirm new phone | SIGNED_IN |

**Error Cases:**

- `invalid_otp` - wrong OTP code
- `expired_otp` - OTP expired
- `otp_already_used` - OTP already verified
- `over_request_rate_limit` - too many attempts
- `captcha_verification_failed` - invalid CAPTCHA

**Related:** `signInWithOtp`, `resend`, `resetPasswordForEmail`

---

### `reauthenticate()`

**Priority:** High | **Complexity:** Simple

**Signature:**

```ts
async reauthenticate(): Promise<AuthResponse>
```

**Purpose:** Send reauthentication OTP for sensitive operations (requires active session)

**Parameters:** None

**Returns:**

```ts
{
  data: {
    user: null
    session: null
  }
  error: AuthError | null
}
```

**Usage:**

```ts
// Before sensitive operation (e.g., password change, account deletion)
const { error } = await client.auth.reauthenticate()

if (error) {
  console.error('Reauthentication failed:', error.message)
} else {
  console.log('Reauthentication OTP sent to your email/phone')

  // User receives OTP, then verify it
  const otp = prompt('Enter OTP:')
  const { data, error: verifyError } = await client.auth.verifyOtp({
    email: currentUser.email,
    token: otp,
    type: 'email',
  })

  if (!verifyError) {
    // Now perform sensitive operation
    await client.auth.updateUser({ password: 'newPassword' })
  }
}
```

**Implementation Considerations:**

- **Backend Requirements:** OTP generation, email/SMS delivery, active session validation
- **Security:** Ensures user recently authenticated before sensitive operations (reduces session hijacking risk)
- **Complexity:** Simple - OTP generation + delivery
- **Dependencies:** Email/SMS service, active session
- **Storage:** Does not modify storage (no session created)

**Authentication Flow:**

```
reauthenticate()
  → Get current session JWT
  → GET /reauthenticate
      → Headers: Authorization: Bearer <jwt>
  → Backend:
      → Validate JWT
      → Lookup user by JWT sub claim
      → Generate OTP
      → Send OTP to user's registered email or phone
      → Store OTP hash
  → Client: Return success (no session)
  → User verifies OTP via verifyOtp()
  → Perform sensitive operation
```

**GoTrue Backend Mapping:**

- **Endpoint:** `GET /reauthenticate`
- **Headers:** `Authorization: Bearer <jwt>`
- **Response:** 204 No Content (success, OTP sent)

**Use Cases:**

- Before password change
- Before email/phone change
- Before account deletion
- Before viewing sensitive data
- Before linking new identity

**Example: Password Change with Reauthentication**

```ts
async function changePassword(newPassword: string) {
  // Step 1: Request reauthentication OTP
  const { error: reauth Error } = await client.auth.reauthenticate()
  if (reauthError) throw reauthError

  // Step 2: Prompt user for OTP
  const otp = prompt('Enter OTP from email:')

  // Step 3: Verify OTP
  const { error: verifyError } = await client.auth.verifyOtp({
    email: currentUser.email,
    token: otp,
    type: 'email',
  })
  if (verifyError) throw verifyError

  // Step 4: Now update password
  const { error: updateError } = await client.auth.updateUser({
    password: newPassword,
  })
  if (updateError) throw updateError

  console.log('Password changed successfully')
}
```

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `unauthorized` - invalid JWT
- `over_email_send_rate_limit` / `over_sms_send_rate_limit` - too many requests

**Related:** `verifyOtp`, `updateUser`

---

## Summary

**13 HIGH priority methods documented:**

- ✅ Session Management (5): getSession, setSession, refreshSession, getUser, initialize
- ✅ Sign In (4): signInWithPassword, signInWithOAuth, signInWithOtp, exchangeCodeForSession
- ✅ Sign Up (1): signUp
- ✅ Sign Out (1): signOut
- ✅ Verification (2): verifyOtp, reauthenticate

**Key Implementation Patterns:**

1. **Session Storage:** All auth methods save session to storage (localStorage or custom)
2. **Event Notification:** Methods notify subscribers (SIGNED_IN, SIGNED_OUT, PASSWORD_RECOVERY)
3. **Error Handling:** Return { data, error } - never throw (except programming errors)
4. **PKCE Support:** OAuth and email OTP support PKCE flow for enhanced security
5. **Lock Mechanism:** Session operations use locks to prevent race conditions
6. **Auto-Refresh:** Session auto-refreshed when expired (in getSession, setSession)

**Common Error Types:**

- `AuthSessionMissingError` - no active session
- `AuthInvalidCredentialsError` - wrong credentials or missing required fields
- `AuthInvalidTokenResponseError` - incomplete API response
- `AuthPKCECodeVerifierMissingError` - PKCE verifier missing
- `invalid_grant` - refresh token invalid
- `invalid_credentials` - wrong email/password
- `over_request_rate_limit` - rate limit exceeded

**Security Considerations:**

- Passwords hashed with bcrypt
- PKCE flow prevents authorization code interception
- Refresh tokens are single-use (rotation)
- Access tokens cannot be revoked (short expiry recommended)
- Use `getUser()` for authorization decisions (not `getSession()`)
- CAPTCHA support for bot protection
- Rate limiting on sensitive endpoints
