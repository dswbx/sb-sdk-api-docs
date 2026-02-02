# User Management

**Priority:** MEDIUM
**Methods:** 5
**Status:** Complete

Profile updates, password recovery, resend confirmations, identity queries, auth state subscriptions.

---

## Methods

### `updateUser(attributes, options?)`

**Priority:** Medium | **Complexity:** Moderate

**Signature:**

```ts
async updateUser(
  attributes: UserAttributes,
  options: {
    emailRedirectTo?: string
  } = {}
): Promise<UserResponse>
```

**Purpose:** Update user profile (email, phone, password, metadata)

**Parameters:**

- `attributes.email?` (string) - New email address (requires confirmation)
- `attributes.phone?` (string) - New phone number (requires confirmation)
- `attributes.password?` (string) - New password
- `attributes.nonce?` (string) - Nonce from reauthenticate() for password change
- `attributes.data?` (object) - Custom user metadata
- `options.emailRedirectTo?` (string) - URL to redirect after email confirmation

**Returns:**

```ts
{ data: { user: User }, error: AuthError | null }
```

**Usage:**

```ts
// Update user metadata
const { data, error } = await client.auth.updateUser({
  data: { name: 'John Doe', age: 30 },
})

// Change email (requires confirmation)
const { data, error } = await client.auth.updateUser({
  email: 'newemail@example.com',
})
// User receives confirmation email
// After clicking link, email is updated

// Change password
const { data, error } = await client.auth.updateUser({
  password: 'newPassword123!',
})

// Change phone (requires OTP)
const { data, error } = await client.auth.updateUser({
  phone: '+12025559999',
})
// User receives SMS with OTP
// Must verify OTP to complete

// Change password with reauthentication
await client.auth.reauthenticate()
const otp = prompt('Enter OTP:')
await client.auth.verifyOtp({ email: user.email, token: otp, type: 'email' })
await client.auth.updateUser({
  password: 'newPassword123!',
  nonce: 'nonce_from_reauthenticate',
})

// Multiple updates
const { data, error } = await client.auth.updateUser({
  data: { theme: 'dark', notifications: true },
  email: 'newemail@example.com',
})
```

**Implementation Considerations:**

- **Backend Requirements:** User update endpoint, JWT validation, email/SMS delivery (for changes)
- **Security:** Email/phone changes require verification, password changes recommended with reauthentication
- **Complexity:** Moderate - validation + confirmation flow for sensitive fields
- **Dependencies:** Email service (email change), SMS provider (phone change)
- **Storage:** Updates session user object, notifies USER_UPDATED event

**Authentication Flow:**

```
updateUser({ email, password, data })
  → PUT /user
      → Headers: Authorization: Bearer <jwt>
      → Body: { email, password, data, code_challenge, code_challenge_method }
  → Backend:
      → Validate JWT
      → If password: hash and update
      → If email: send confirmation email (keep old email until confirmed)
      → If phone: send SMS OTP (keep old phone until verified)
      → If data: merge into user_metadata
      → Return updated user object
  → Client: Update session storage
  → Notify subscribers (USER_UPDATED)
```

**GoTrue Backend Mapping:**

- **Endpoint:** `PUT /user`
- **Headers:** `Authorization: Bearer <jwt>`
- **Request Body:**

```json
{
  "email": "newemail@example.com",
  "password": "newPassword123!",
  "phone": "+12025559999",
  "data": {
    "name": "John Doe"
  },
  "nonce": "reauthenticate_nonce",
  "code_challenge": "...",
  "code_challenge_method": "S256"
}
```

- **Response:**

```json
{
  "id": "uuid",
  "email": "user@example.com",
  "new_email": "newemail@example.com",
  "phone": "+12025551234",
  "new_phone": "+12025559999",
  "user_metadata": { "name": "John Doe" },
  "updated_at": "2024-01-01T00:00:00Z",
  ...
}
```

**Email/Phone Change Flow:**

```
# Email Change
updateUser({ email: 'new@example.com' })
  → Server sends confirmation to new@example.com
  → User clicks link with token_hash
  → verifyOtp({ token_hash, type: 'email_change' })
  → new_email → email, new_email cleared

# Phone Change
updateUser({ phone: '+19999999999' })
  → Server sends OTP to new phone
  → verifyOtp({ phone: '+19999999999', token: '123456', type: 'phone_change' })
  → new_phone → phone, new_phone cleared
```

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `email_exists` - email already taken
- `phone_exists` - phone already taken
- `weak_password` - password doesn't meet requirements
- `unauthorized` - invalid JWT

**Related:** `getUser`, `reauthenticate`, `verifyOtp`

---

### `resetPasswordForEmail(email, options?)`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async resetPasswordForEmail(
  email: string,
  options: {
    redirectTo?: string
    captchaToken?: string
  } = {}
): Promise<
  | { data: {}; error: null }
  | { data: null; error: AuthError }
>
```

**Purpose:** Send password reset email with magic link

**Parameters:**

- `email` (string) - User's email address
- `options.redirectTo?` (string) - URL to redirect after clicking reset link
- `options.captchaToken?` (string) - CAPTCHA token

**Returns:**

```ts
{ data: {}, error: AuthError | null }
```

**Usage:**

```ts
// Basic password reset
const { error } = await client.auth.resetPasswordForEmail('user@example.com')

if (!error) {
  console.log('Password reset email sent')
}

// With redirect URL
const { error } = await client.auth.resetPasswordForEmail(
  'user@example.com',
  {
    redirectTo: 'https://myapp.com/reset-password',
  }
)

// With CAPTCHA
const { error } = await client.auth.resetPasswordForEmail(
  'user@example.com',
  {
    captchaToken: 'recaptcha_token...',
  }
)

// Complete flow:
// 1. Request reset
await client.auth.resetPasswordForEmail('user@example.com')

// 2. User clicks email link, redirected to app with token_hash
// 3. App extracts token from URL
const url = new URL(window.location.href)
const token_hash = url.searchParams.get('token_hash')

// 4. Verify token (creates PASSWORD_RECOVERY session)
await client.auth.verifyOtp({
  token_hash,
  type: 'recovery',
})

// 5. Update password
await client.auth.updateUser({
  password: 'newPassword123!',
})
```

**Implementation Considerations:**

- **Backend Requirements:** Email service, token generation, token storage
- **Security:** Token expires (typically 1 hour), single-use, PKCE support, doesn't reveal account existence
- **Complexity:** Simple - token generation + email delivery
- **Dependencies:** Email service
- **Storage:** Stores PKCE code verifier (cleared after verification or error)

**Authentication Flow:**

```
resetPasswordForEmail('user@example.com')
  → POST /recover
      → Body: { email, code_challenge, code_challenge_method, captcha_token }
  → Backend:
      → Lookup user by email (fail silently if not found - security)
      → Generate recovery token
      → Send email with recovery link
      → Link: redirectTo?token_hash=...&type=recovery
  → Client: Success (no indication if user exists)
  → User clicks link
  → verifyOtp({ token_hash, type: 'recovery' })
  → Special PASSWORD_RECOVERY session (short-lived)
  → updateUser({ password: 'new' })
  → Password updated
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /recover`
- **Request Body:**

```json
{
  "email": "user@example.com",
  "code_challenge": "...",
  "code_challenge_method": "S256",
  "gotrue_meta_security": {
    "captcha_token": "..."
  }
}
```

- **Response:** 200 OK (no body, doesn't reveal if user exists)

**Email Template:**

```
Subject: Reset Your Password

Click the link below to reset your password:
{{ .ConfirmationURL }}?token_hash={{ .TokenHash }}&type=recovery

This link expires in 1 hour.
```

**Error Cases:**

- `over_email_send_rate_limit` - too many reset requests
- `captcha_verification_failed` - invalid CAPTCHA
- Note: Does NOT return error if email not found (security)

**Related:** `verifyOtp`, `updateUser`, `reauthenticate`

---

### `resend(credentials)`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async resend(credentials: ResendParams): Promise<AuthOtpResponse>

// Params type
ResendParams =
  | {
      type: 'signup' | 'email_change'
      email: string
      options?: {
        emailRedirectTo?: string
        captchaToken?: string
      }
    }
  | {
      type: 'sms' | 'phone_change'
      phone: string
      options?: {
        captchaToken?: string
      }
    }
```

**Purpose:** Resend confirmation email or SMS OTP

**Parameters:**

- **Email resend:**
  - `credentials.type` ('signup' | 'email_change')
  - `credentials.email` (string)
  - `credentials.options.emailRedirectTo?` (string)
  - `credentials.options.captchaToken?` (string)
- **Phone resend:**
  - `credentials.type` ('sms' | 'phone_change')
  - `credentials.phone` (string)
  - `credentials.options.captchaToken?` (string)

**Returns:**

```ts
{
  data: {
    user: null
    session: null
    messageId?: string  // Phone only
  }
  error: AuthError | null
}
```

**Usage:**

```ts
// Resend signup confirmation email
const { error } = await client.auth.resend({
  type: 'signup',
  email: 'user@example.com',
})

// Resend email change confirmation
const { error } = await client.auth.resend({
  type: 'email_change',
  email: 'newemail@example.com',
  options: {
    emailRedirectTo: 'https://myapp.com/confirm',
  },
})

// Resend SMS OTP (signup)
const { data, error } = await client.auth.resend({
  type: 'sms',
  phone: '+12025551234',
})
console.log('Message ID:', data?.messageId)

// Resend phone change OTP
const { error } = await client.auth.resend({
  type: 'phone_change',
  phone: '+12025559999',
})

// With CAPTCHA
const { error } = await client.auth.resend({
  type: 'signup',
  email: 'user@example.com',
  options: {
    captchaToken: 'recaptcha_token...',
  },
})
```

**Implementation Considerations:**

- **Backend Requirements:** Email/SMS service, token lookup, rate limiting
- **Security:** Rate limited, single-use tokens, expires (typically 1 hour)
- **Complexity:** Simple - lookup existing token context + resend
- **Dependencies:** Email service (email types), SMS provider (phone types)
- **Storage:** No storage operations

**Authentication Flow:**

```
resend({ type: 'signup', email: 'user@example.com' })
  → POST /resend
      → Body: { type, email/phone, captcha_token }
  → Backend:
      → Lookup user by email/phone
      → Check rate limit
      → If type=signup: resend signup confirmation
      → If type=email_change: resend email change confirmation
      → If type=sms: resend signup SMS OTP
      → If type=phone_change: resend phone change OTP
      → Generate new token (invalidate old)
      → Send email/SMS
  → Client: Success (user must verify)
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /resend`
- **Request Body (Email):**

```json
{
  "type": "signup",
  "email": "user@example.com",
  "gotrue_meta_security": {
    "captcha_token": "..."
  }
}
```

- **Request Body (Phone):**

```json
{
  "type": "sms",
  "phone": "+12025551234",
  "gotrue_meta_security": {
    "captcha_token": "..."
  }
}
```

- **Response (Phone):**

```json
{
  "message_id": "SM1234..."
}
```

**Resend Types:**

| Type | Use Case | User State | Verification Method |
|------|----------|------------|---------------------|
| `signup` | Resend signup confirmation | Unconfirmed | verifyOtp with type=signup |
| `email_change` | Resend email change confirmation | new_email set | verifyOtp with type=email_change |
| `sms` | Resend signup SMS OTP | Unconfirmed phone | verifyOtp with type=sms |
| `phone_change` | Resend phone change OTP | new_phone set | verifyOtp with type=phone_change |

**Error Cases:**

- `AuthInvalidCredentialsError` - neither email nor phone provided
- `over_email_send_rate_limit` - too many email resends
- `over_sms_send_rate_limit` - too many SMS resends
- `user_not_found` - user doesn't exist
- `captcha_verification_failed` - invalid CAPTCHA

**Related:** `signUp`, `signInWithOtp`, `updateUser`, `verifyOtp`

---

### `getUserIdentities()`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async getUserIdentities(): Promise<
  | { data: { identities: UserIdentity[] }; error: null }
  | { data: null; error: AuthError }
>
```

**Purpose:** List all OAuth/OIDC identities linked to user account

**Parameters:** None

**Returns:**

```ts
{
  data: {
    identities: UserIdentity[]
  }
  error: AuthError | null
}

// UserIdentity type
{
  id: string
  user_id: string
  identity_data: {
    email?: string
    email_verified?: boolean
    phone_verified?: boolean
    sub?: string
    [key: string]: any
  }
  provider: string  // 'email', 'google', 'github', etc.
  last_sign_in_at: string
  created_at: string
  updated_at: string
}
```

**Usage:**

```ts
// Get all linked identities
const { data, error } = await client.auth.getUserIdentities()

if (data) {
  console.log('Identities:', data.identities)

  data.identities.forEach(identity => {
    console.log(`- ${identity.provider}: ${identity.identity_data.email}`)
  })
}

// Check if specific provider linked
const { data } = await client.auth.getUserIdentities()
const hasGoogle = data?.identities.some(i => i.provider === 'google')

if (!hasGoogle) {
  // Offer to link Google account
  await client.auth.linkIdentity({ provider: 'google' })
}

// List all providers
const { data } = await client.auth.getUserIdentities()
const providers = data?.identities.map(i => i.provider) || []
console.log('Linked providers:', providers)

// Get identity for unlinking
const { data } = await client.auth.getUserIdentities()
const githubIdentity = data?.identities.find(i => i.provider === 'github')

if (githubIdentity) {
  await client.auth.unlinkIdentity(githubIdentity)
}
```

**Implementation Considerations:**

- **Backend Requirements:** User fetch endpoint (calls getUser internally)
- **Security:** Requires active session
- **Complexity:** Simple - reads from user.identities
- **Dependencies:** None
- **Storage:** No storage operations

**Authentication Flow:**

```
getUserIdentities()
  → Calls getUser() internally
  → GET /user
      → Headers: Authorization: Bearer <jwt>
  → Backend: Returns user with identities array
  → Extract user.identities
  → Return { identities }
```

**Identity Data Structure:**

```json
{
  "identities": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "identity_data": {
        "email": "user@example.com",
        "email_verified": true,
        "sub": "google_user_id"
      },
      "provider": "email",
      "last_sign_in_at": "2024-01-01T00:00:00Z",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    },
    {
      "id": "uuid",
      "user_id": "uuid",
      "identity_data": {
        "email": "user@gmail.com",
        "email_verified": true,
        "sub": "google_123456"
      },
      "provider": "google",
      "last_sign_in_at": "2024-01-02T00:00:00Z",
      "created_at": "2024-01-02T00:00:00Z",
      "updated_at": "2024-01-02T00:00:00Z"
    }
  ]
}
```

**Provider Types:**

- **Built-in:** `email`, `phone`, `anonymous`
- **OAuth:** `google`, `github`, `apple`, `azure`, `facebook`, `twitter`, etc.
- **SAML:** Custom SAML providers

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `unauthorized` - invalid JWT
- Returns empty array if no identities (not an error)

**Related:** `linkIdentity`, `unlinkIdentity`, `getUser`

---

### `onAuthStateChange(callback)`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
onAuthStateChange(
  callback: (event: AuthChangeEvent, session: Session | null) => void
): { data: { subscription: Subscription } }

// Async callback (deprecated - risk of deadlock)
onAuthStateChange(
  callback: (event: AuthChangeEvent, session: Session | null) => Promise<void>
): { data: { subscription: Subscription } }

// Event types
type AuthChangeEvent =
  | 'INITIAL_SESSION'
  | 'SIGNED_IN'
  | 'SIGNED_OUT'
  | 'TOKEN_REFRESHED'
  | 'USER_UPDATED'
  | 'PASSWORD_RECOVERY'
  | 'MFA_CHALLENGE_VERIFIED'
```

**Purpose:** Subscribe to auth events (sign in, sign out, token refresh, etc.)

**Parameters:**

- `callback` (function) - Called on every auth event. Receives event name and session.

**Returns:**

```ts
{
  data: {
    subscription: {
      id: string | symbol
      callback: Function
      unsubscribe: () => void
    }
  }
}
```

**Usage:**

```ts
// Basic subscription
const { data: { subscription } } = client.auth.onAuthStateChange((event, session) => {
  console.log('Auth event:', event)

  if (event === 'SIGNED_IN') {
    console.log('User signed in:', session?.user.email)
  } else if (event === 'SIGNED_OUT') {
    console.log('User signed out')
    // Redirect to login
  }
})

// Unsubscribe when done
subscription.unsubscribe()

// React example
useEffect(() => {
  const { data: { subscription } } = client.auth.onAuthStateChange(
    (event, session) => {
      setUser(session?.user ?? null)
    }
  )

  return () => subscription.unsubscribe()
}, [])

// Handle all events
client.auth.onAuthStateChange((event, session) => {
  switch (event) {
    case 'INITIAL_SESSION':
      console.log('Initial session loaded')
      break
    case 'SIGNED_IN':
      console.log('User signed in')
      break
    case 'SIGNED_OUT':
      console.log('User signed out')
      window.location.href = '/login'
      break
    case 'TOKEN_REFRESHED':
      console.log('Token refreshed')
      break
    case 'USER_UPDATED':
      console.log('User updated')
      break
    case 'PASSWORD_RECOVERY':
      console.log('Password recovery')
      window.location.href = '/reset-password'
      break
    case 'MFA_CHALLENGE_VERIFIED':
      console.log('MFA verified')
      break
  }
})

// Multiple subscriptions (all fire)
const sub1 = client.auth.onAuthStateChange((event, session) => {
  console.log('Subscription 1:', event)
})

const sub2 = client.auth.onAuthStateChange((event, session) => {
  console.log('Subscription 2:', event)
})

// Unsubscribe both
sub1.data.subscription.unsubscribe()
sub2.data.subscription.unsubscribe()
```

**Implementation Considerations:**

- **Backend Requirements:** None - client-side only
- **Security:** Callbacks run inside exclusive lock - AVOID async callbacks (deadlock risk)
- **Complexity:** Simple - event emitter pattern
- **Dependencies:** None
- **Storage:** No storage operations

**Event Emission Flow:**

```
onAuthStateChange(callback)
  → Generate unique callback ID
  → Store callback in stateChangeEmitters map
  → Emit INITIAL_SESSION immediately with current session
  → Return subscription object

When auth events occur:
  → _notifyAllSubscribers(event, session)
      → Acquire lock
      → Iterate stateChangeEmitters
      → Call each callback(event, session)
      → Release lock
```

**Auth Events:**

| Event | When Fired | Session State |
|-------|-----------|---------------|
| `INITIAL_SESSION` | On subscription | Current session or null |
| `SIGNED_IN` | signInWithPassword, signUp, verifyOtp, OAuth callback | Session present |
| `SIGNED_OUT` | signOut (scope=global/local) | null |
| `TOKEN_REFRESHED` | Auto-refresh or manual refreshSession | Updated session |
| `USER_UPDATED` | updateUser, linkIdentity | Updated session |
| `PASSWORD_RECOVERY` | verifyOtp with type=recovery | Recovery session |
| `MFA_CHALLENGE_VERIFIED` | mfa.verify | Elevated session (aal2) |

**Cross-Tab Synchronization:**

- Uses BroadcastChannel for cross-tab sync (browsers)
- All tabs receive auth events
- Useful for multi-tab logout

**Async Callback Warning:**

```ts
// ❌ AVOID - Risk of deadlock
client.auth.onAuthStateChange(async (event, session) => {
  await someAsyncOperation()  // If this calls another auth method → deadlock
})

// ✅ RECOMMENDED - Synchronous callback
client.auth.onAuthStateChange((event, session) => {
  // Dispatch to async handler outside lock
  handleAuthEvent(event, session)
})

async function handleAuthEvent(event, session) {
  // Async operations safe here
  await updateUI()
  await client.auth.getUser()  // No deadlock
}
```

**Error Cases:**

- None - always succeeds

**Related:** All auth methods (emit events)

---

## Summary

**5 user management methods documented:**

- ✅ updateUser - Update profile, email, phone, password, metadata
- ✅ resetPasswordForEmail - Send password reset email
- ✅ resend - Resend confirmation email/SMS
- ✅ getUserIdentities - List linked OAuth identities
- ✅ onAuthStateChange - Subscribe to auth events

**Key Patterns:**

1. **Session Required:** updateUser, getUserIdentities require active session
2. **Confirmation Flows:** Email/phone changes require verification via verifyOtp
3. **Rate Limiting:** resetPasswordForEmail, resend are rate limited
4. **Event Notification:** updateUser emits USER_UPDATED, onAuthStateChange receives all events
5. **PKCE Support:** resetPasswordForEmail supports PKCE for email recovery

**Common Use Cases:**

- **Profile updates:** updateUser({ data: {...} })
- **Password reset:** resetPasswordForEmail → verifyOtp → updateUser
- **Email change:** updateUser({ email }) → resend (if needed) → verifyOtp
- **Auth state sync:** onAuthStateChange for UI updates
- **Identity management:** getUserIdentities → unlinkIdentity
