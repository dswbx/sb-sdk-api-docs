# Auth API Overview

**Total Methods:** ~70
**Documented:** 18/70 (25.7%)

Complete catalog of all auth-js methods organized by priority and category.

---

## Quick Reference by Priority

### HIGH Priority (13 methods) - 80% Usage

Core authentication flows used by most applications.

| Method | Category | Complexity | Status |
|--------|----------|------------|--------|
| `getSession()` | Session | Simple | ✅ Documented |
| `setSession()` | Session | Moderate | ✅ Documented |
| `refreshSession()` | Session | Simple | ✅ Documented |
| `getUser()` | Session | Simple | ✅ Documented |
| `initialize()` | Session | Moderate | ✅ Documented |
| `signInWithPassword()` | Sign In | Simple | ✅ Documented |
| `signInWithOAuth()` | Sign In | Moderate | ✅ Documented |
| `signInWithOtp()` | Sign In | Moderate | ✅ Documented |
| `exchangeCodeForSession()` | Sign In | Moderate | ✅ Documented |
| `signUp()` | Sign Up | Simple | ✅ Documented |
| `signOut()` | Sign Out | Simple | ✅ Documented |
| `verifyOtp()` | Verification | Simple | ✅ Documented |
| `reauthenticate()` | Verification | Simple | ✅ Documented |

### MEDIUM Priority (25 methods) - 15% Usage

Advanced features for enhanced functionality.

| Method | Category | Complexity | Status |
|--------|----------|------------|--------|
| `updateUser()` | User Management | Moderate | ✅ Documented |
| `resetPasswordForEmail()` | User Management | Simple | ✅ Documented |
| `resend()` | User Management | Simple | ✅ Documented |
| `getUserIdentities()` | User Management | Simple | ✅ Documented |
| `onAuthStateChange()` | User Management | Simple | ✅ Documented |
| `linkIdentity()` | Identity | Moderate | Planned |
| `unlinkIdentity()` | Identity | Simple | Planned |
| `mfa.enroll()` | MFA | Moderate | Planned |
| `mfa.challenge()` | MFA | Moderate | Planned |
| `mfa.verify()` | MFA | Moderate | Planned |
| `mfa.unenroll()` | MFA | Simple | Planned |
| `mfa.listFactors()` | MFA | Simple | Planned |
| `mfa.challengeAndVerify()` | MFA | Moderate | Planned |
| `mfa.getAuthenticatorAssuranceLevel()` | MFA | Simple | Planned |
| `mfa.webauthn.enroll()` | MFA/WebAuthn | Complex | Planned |
| `mfa.webauthn.challenge()` | MFA/WebAuthn | Complex | Planned |
| `mfa.webauthn.verify()` | MFA/WebAuthn | Complex | Planned |
| `mfa.webauthn.authenticate()` | MFA/WebAuthn | Complex | Planned |
| `mfa.webauthn.register()` | MFA/WebAuthn | Complex | Planned |
| `signInWithSSO()` | Advanced Auth | Complex | Planned |
| `signInWithIdToken()` | Advanced Auth | Moderate | Planned |
| `signInWithWeb3()` | Advanced Auth | Complex | Planned |
| `signInAnonymously()` | Advanced Auth | Simple | Planned |
| `startAutoRefresh()` | Advanced Auth | Simple | Planned |
| `stopAutoRefresh()` | Advanced Auth | Simple | Planned |

### LOW Priority (30+ methods) - 5% Usage

Admin, OAuth server, and specialized operations.

| Method | Category | Complexity | Status |
|--------|----------|------------|--------|
| `admin.createUser()` | Admin | Simple | Planned |
| `admin.listUsers()` | Admin | Simple | Planned |
| `admin.getUserById()` | Admin | Simple | Planned |
| `admin.updateUserById()` | Admin | Simple | Planned |
| `admin.deleteUser()` | Admin | Simple | Planned |
| `admin.inviteUserByEmail()` | Admin | Simple | Planned |
| `admin.generateLink()` | Admin | Simple | Planned |
| `admin.signOut()` | Admin | Simple | Planned |
| `admin.mfa.listFactors()` | Admin/MFA | Simple | Planned |
| `admin.mfa.deleteFactor()` | Admin/MFA | Simple | Planned |
| `oauth.getAuthorizationDetails()` | OAuth Server | Moderate | Planned |
| `oauth.approveAuthorization()` | OAuth Server | Moderate | Planned |
| `oauth.denyAuthorization()` | OAuth Server | Moderate | Planned |
| `oauth.listGrants()` | OAuth Server | Simple | Planned |
| `oauth.revokeGrant()` | OAuth Server | Simple | Planned |
| `oauth.listClients()` | OAuth Admin | Simple | Planned |
| `oauth.createClient()` | OAuth Admin | Simple | Planned |
| `oauth.getClient()` | OAuth Admin | Simple | Planned |
| `oauth.updateClient()` | OAuth Admin | Simple | Planned |
| `oauth.deleteClient()` | OAuth Admin | Simple | Planned |
| `oauth.regenerateClientSecret()` | OAuth Admin | Simple | Planned |
| `getClaims()` | Utilities | Moderate | Planned |
| `isThrowOnErrorEnabled()` | Utilities | Simple | Planned |

---

## Categorization by Function

### Session Management (5 methods)

Session lifecycle, token refresh, user retrieval.

- **getSession()** - Get current session from storage, auto-refresh if expired
- **setSession()** - Set session from tokens, validate via network
- **refreshSession()** - Force token refresh regardless of expiry
- **getUser()** - Get authenticated user via network (for authorization)
- **initialize()** - Initialize from URL callback or storage

### Sign In (4 methods)

User authentication flows.

- **signInWithPassword()** - Email/phone + password authentication
- **signInWithOAuth()** - OAuth provider (Google, GitHub, Apple, etc.)
- **signInWithOtp()** - Passwordless magic link or OTP
- **exchangeCodeForSession()** - PKCE code exchange (completes OAuth/OTP)

### Sign Up (1 method)

New user registration.

- **signUp()** - Create account with email/phone + password

### Sign Out (1 method)

Session termination.

- **signOut()** - Revoke tokens, clear session (global/local/others scope)

### Verification (2 methods)

Token and OTP verification.

- **verifyOtp()** - Verify email/SMS OTP, create session
- **reauthenticate()** - Send OTP for sensitive operations

### User Management (5 methods)

Profile updates, password recovery, state changes.

- **updateUser()** - Update email, phone, password, metadata
- **resetPasswordForEmail()** - Send password reset link
- **resend()** - Resend confirmation email/SMS
- **getUserIdentities()** - List linked OAuth identities
- **onAuthStateChange()** - Subscribe to auth events

### Identity Linking (2 methods)

OAuth account linking.

- **linkIdentity()** - Link OAuth/OIDC provider to account
- **unlinkIdentity()** - Remove linked identity

### Multi-Factor Authentication (12 methods)

TOTP, SMS, WebAuthn MFA.

**Core MFA (7):**
- **mfa.enroll()** - Create new MFA factor (TOTP/phone/WebAuthn)
- **mfa.challenge()** - Generate challenge for verification
- **mfa.verify()** - Verify challenge with code/credential
- **mfa.unenroll()** - Remove MFA factor
- **mfa.listFactors()** - List user's MFA factors
- **mfa.challengeAndVerify()** - Combined challenge + verify
- **mfa.getAuthenticatorAssuranceLevel()** - Get AAL (aal1/aal2)

**WebAuthn (5):**
- **mfa.webauthn.enroll()** - Enroll WebAuthn credential
- **mfa.webauthn.challenge()** - Get WebAuthn challenge + create/request credential
- **mfa.webauthn.verify()** - Verify WebAuthn credential
- **mfa.webauthn.authenticate()** - One-step authentication (challenge + verify)
- **mfa.webauthn.register()** - One-step registration (enroll + challenge + verify)

### Advanced Authentication (6 methods)

Enterprise and specialized auth.

- **signInWithSSO()** - SAML/OIDC SSO (enterprise)
- **signInWithIdToken()** - OIDC ID token authentication
- **signInWithWeb3()** - Ethereum/Solana wallet (SIWE/SIWS)
- **signInAnonymously()** - Anonymous user creation
- **startAutoRefresh()** - Manual token refresh control
- **stopAutoRefresh()** - Stop auto-refresh

### Admin API (14+ methods)

Server-side user management.

**User CRUD:**
- **admin.createUser()** - Create user (skip email confirmation)
- **admin.listUsers()** - List all users (paginated)
- **admin.getUserById()** - Get user by ID
- **admin.updateUserById()** - Update user
- **admin.deleteUser()** - Delete user

**Invites & Links:**
- **admin.inviteUserByEmail()** - Send invite email
- **admin.generateLink()** - Generate magic link

**Admin Operations:**
- **admin.signOut()** - Sign out user by JWT
- **admin.mfa.listFactors()** - List user MFA factors
- **admin.mfa.deleteFactor()** - Delete user MFA factor

### OAuth 2.1 Server (5 methods)

Authorization server implementation.

- **oauth.getAuthorizationDetails()** - Get OAuth authorization request details
- **oauth.approveAuthorization()** - Approve OAuth consent
- **oauth.denyAuthorization()** - Deny OAuth consent
- **oauth.listGrants()** - List user's OAuth grants
- **oauth.revokeGrant()** - Revoke OAuth grant

### OAuth Admin (6 methods)

OAuth client configuration.

- **oauth.listClients()** - List OAuth clients
- **oauth.createClient()** - Create OAuth client
- **oauth.getClient()** - Get OAuth client
- **oauth.updateClient()** - Update OAuth client
- **oauth.deleteClient()** - Delete OAuth client
- **oauth.regenerateClientSecret()** - Regenerate client secret

### Utilities (2 methods)

JWT and configuration helpers.

- **getClaims()** - Verify JWT offline with asymmetric keys
- **isThrowOnErrorEnabled()** - Check error handling mode

---

## Common Workflows

### Password Authentication Flow

```
signUp({ email, password })
  → User receives confirmation email (if required)
  → verifyOtp({ token_hash, type: 'signup' })
  → signInWithPassword({ email, password })
  → getSession() returns session
```

### OAuth Flow

```
signInWithOAuth({ provider: 'google' })
  → Browser redirects to Google
  → User approves
  → Redirect back with code
  → initialize() calls exchangeCodeForSession(code)
  → Session created
```

### Magic Link Flow

```
signInWithOtp({ email })
  → User receives email with link
  → User clicks link
  → initialize() detects token_hash
  → verifyOtp({ token_hash, type: 'magiclink' })
  → Session created
```

### Password Reset Flow

```
resetPasswordForEmail('user@example.com')
  → User receives reset email
  → User clicks link
  → verifyOtp({ token_hash, type: 'recovery' })
  → PASSWORD_RECOVERY session
  → updateUser({ password: 'new' })
  → Password updated
```

### Email Change Flow

```
updateUser({ email: 'new@example.com' })
  → User receives confirmation at new email
  → verifyOtp({ token_hash, type: 'email_change' })
  → new_email → email
```

### MFA Enrollment Flow

```
mfa.enroll({ factorType: 'totp', friendlyName: 'My App' })
  → User scans QR code in authenticator app
  → mfa.challenge({ factorId })
  → User enters code from app
  → mfa.verify({ factorId, challengeId, code })
  → Factor verified, AAL elevated to aal2
```

### MFA Authentication Flow

```
signInWithPassword({ email, password })
  → User authenticated (aal1)
  → mfa.challenge({ factorId })
  → User enters MFA code
  → mfa.verify({ factorId, challengeId, code })
  → AAL elevated to aal2
```

---

## Authentication Events

Events emitted via `onAuthStateChange()`:

| Event | Trigger | Session State | Use Case |
|-------|---------|---------------|----------|
| `INITIAL_SESSION` | Subscription created | Current or null | Initialize UI on mount |
| `SIGNED_IN` | Sign in methods, verifyOtp | Present | Show authenticated UI |
| `SIGNED_OUT` | signOut (global/local) | null | Redirect to login |
| `TOKEN_REFRESHED` | Auto/manual refresh | Updated | Update stored session |
| `USER_UPDATED` | updateUser, linkIdentity | Updated | Refresh user display |
| `PASSWORD_RECOVERY` | verifyOtp type=recovery | Recovery session | Show password form |
| `MFA_CHALLENGE_VERIFIED` | mfa.verify | Elevated (aal2) | Allow sensitive ops |

---

## Error Patterns

### Common Errors Across Methods

**Session Errors:**
- `AuthSessionMissingError` - No active session (call signIn first)

**Validation Errors:**
- `AuthInvalidCredentialsError` - Missing required fields
- `invalid_credentials` - Wrong email/password
- `invalid_grant` - Refresh token invalid/expired

**Rate Limiting:**
- `over_request_rate_limit` - Too many requests
- `over_email_send_rate_limit` - Too many emails
- `over_sms_send_rate_limit` - Too many SMS

**Verification Errors:**
- `invalid_otp` - Wrong OTP code
- `expired_otp` - OTP expired
- `otp_already_used` - OTP already verified

**User Errors:**
- `user_already_exists` - Email/phone taken
- `user_not_found` - User doesn't exist
- `email_not_confirmed` - Email not verified
- `phone_not_confirmed` - Phone not verified

**PKCE Errors:**
- `AuthPKCECodeVerifierMissingError` - Code verifier missing
- `invalid_request` - Code verifier mismatch

---

## Implementation Notes

### PKCE Support

Methods with PKCE flow support:
- signInWithOAuth
- signInWithOtp (email only)
- signUp (email only)
- resetPasswordForEmail
- exchangeCodeForSession
- linkIdentity (OAuth)

### Session Storage

All sign-in methods save session to storage:
- localStorage (browser default)
- Custom storage adapter (server-side)

Storage key format: `${storageKey}` (default: `sb-{project-ref}-auth-token`)

### Lock Mechanism

Methods using exclusive lock (prevent race conditions):
- getSession
- setSession
- refreshSession
- initialize
- mfa.challenge
- mfa.verify

### Event Notification

Methods that emit events via `onAuthStateChange()`:
- signInWithPassword → SIGNED_IN
- signInWithOAuth → SIGNED_IN (via initialize)
- signUp → SIGNED_IN (if autoconfirm)
- verifyOtp → SIGNED_IN or PASSWORD_RECOVERY
- signOut → SIGNED_OUT
- updateUser → USER_UPDATED
- refreshSession → TOKEN_REFRESHED
- mfa.verify → MFA_CHALLENGE_VERIFIED

---

## Security Considerations

### Passwords
- Hashed with bcrypt (backend)
- Strength validation
- Rate limiting on attempts

### Tokens
- **Access Token (JWT):** Short-lived (1h), cannot be revoked
- **Refresh Token:** Long-lived, single-use (rotation), can be revoked
- **OTP:** Single-use, expires (1h typically)

### PKCE Flow
- Prevents authorization code interception
- Required for public clients (SPAs, mobile)
- Optional for confidential clients (server-side)

### Session Validation
- `getSession()` reads from storage (may be stale/forged)
- `getUser()` makes network request (authentic, use for authorization)

### Rate Limiting
Apply to:
- signInWithPassword (prevent brute force)
- signUp (prevent spam)
- resetPasswordForEmail (prevent enumeration)
- resend (prevent abuse)
- verifyOtp (prevent guessing)

### Email/Phone Confirmation
- Optional but recommended
- Prevents account takeover
- Validates ownership

---

## GoTrue Backend Endpoints

### Authentication
- `POST /signup` - signUp
- `POST /token?grant_type=password` - signInWithPassword
- `POST /token?grant_type=refresh_token` - refreshSession
- `POST /token?grant_type=pkce` - exchangeCodeForSession
- `POST /token?grant_type=id_token` - signInWithIdToken
- `GET /authorize` - signInWithOAuth
- `POST /logout` - signOut

### User Management
- `GET /user` - getUser
- `PUT /user` - updateUser
- `POST /recover` - resetPasswordForEmail
- `POST /verify` - verifyOtp
- `POST /otp` - signInWithOtp
- `POST /resend` - resend
- `GET /reauthenticate` - reauthenticate

### MFA
- `POST /factors` - mfa.enroll
- `POST /factors/{id}/challenge` - mfa.challenge
- `POST /factors/{id}/verify` - mfa.verify
- `DELETE /factors/{id}` - mfa.unenroll

### Identity Linking
- `GET /user/identities/authorize` - linkIdentity
- `DELETE /user/identities/{id}` - unlinkIdentity

### SSO
- `POST /sso` - signInWithSSO

### OAuth Server
- `GET /oauth/authorization` - oauth.getAuthorizationDetails
- `POST /oauth/authorization` - oauth.approveAuthorization
- `DELETE /oauth/authorization` - oauth.denyAuthorization
- `GET /oauth/grants` - oauth.listGrants
- `DELETE /oauth/grants/{id}` - oauth.revokeGrant

### Admin (Requires service role key)
- `POST /admin/users` - admin.createUser
- `GET /admin/users` - admin.listUsers
- `GET /admin/users/{id}` - admin.getUserById
- `PUT /admin/users/{id}` - admin.updateUserById
- `DELETE /admin/users/{id}` - admin.deleteUser
- `POST /admin/generate_link` - admin.generateLink

---

## Type Definitions

### Core Types

```ts
interface Session {
  access_token: string
  refresh_token: string
  expires_in: number
  expires_at: number
  token_type: 'bearer'
  user: User
}

interface User {
  id: string
  email?: string
  phone?: string
  email_confirmed_at?: string
  phone_confirmed_at?: string
  created_at: string
  updated_at: string
  user_metadata: object
  app_metadata: object
  identities?: UserIdentity[]
  factors?: Factor[]
}

interface UserIdentity {
  id: string
  user_id: string
  provider: string
  identity_data: object
  last_sign_in_at: string
  created_at: string
  updated_at: string
}

interface Factor {
  id: string
  friendly_name: string
  factor_type: 'totp' | 'phone' | 'webauthn'
  status: 'verified' | 'unverified'
  created_at: string
  updated_at: string
}
```

### Response Types

```ts
// Standard response
type AuthResponse = {
  data: { user: User | null; session: Session | null }
  error: AuthError | null
}

// User-only response
type UserResponse = {
  data: { user: User }
  error: AuthError | null
}

// OTP response (no session yet)
type AuthOtpResponse = {
  data: { user: null; session: null; messageId?: string }
  error: AuthError | null
}

// OAuth response
type OAuthResponse = {
  data: { provider: Provider; url: string }
  error: AuthError | null
}
```

---

## Documentation Status

- ✅ **HIGH Priority (13/13)** - Complete
- 🔄 **MEDIUM Priority (5/25)** - In Progress
- ⏳ **LOW Priority (0/30+)** - Planned

See [progress.md](./progress.md) for detailed tracking.
