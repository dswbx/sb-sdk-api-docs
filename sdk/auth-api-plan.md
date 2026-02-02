# Auth API Documentation Plan (auth-js / GoTrue)

**Status:** In Progress
**Category:** Auth
**Estimated Methods:** ~70 methods (client + admin + MFA + OAuth)
**Priority:** HIGH - Core authentication functionality

---

## Scope

Document all auth-js (GoTrue) APIs for building custom authentication backend. Focus on understanding functionality, implementation patterns, and integration approaches rather than database compatibility.

**Key Difference from Database Plan:**

- Focus: Implementation guidance, not SQLite compatibility
- Goal: Understand how to build auth features (signup, signin, MFA, etc.)
- Analysis: Implementation complexity, security considerations, integration patterns

---

## Output Structure

Location: `/Users/dennis/Projects/docs/sdk/auth/`

```
auth-api-plan.md                   # THIS FILE - Planning document
progress.md                        # Track completion status
api-overview.md                    # Summary, categorization, quick reference

# HIGH Priority Files [13 methods] - Core Auth Flows
core-authentication.md             # Session, signIn methods, signUp, signOut [0/13]
  - Session: getSession, setSession, refreshSession, getUser, initialize
  - Sign In: signInWithPassword, signInWithOAuth, signInWithOtp, exchangeCodeForSession
  - Sign Up: signUp
  - Sign Out: signOut
  - Verification: verifyOtp, reauthenticate

# MEDIUM Priority Files [~25 methods] - Advanced Features
user-management.md                 # Profile updates, password recovery [0/5]
  - updateUser, resetPasswordForEmail, resend, getUserIdentities
  - State change: onAuthStateChange

identity-linking.md                # OAuth/OIDC identity linking [0/2]
  - linkIdentity, unlinkIdentity

mfa.md                            # Multi-factor authentication [0/12]
  - Enrollment: enroll, webauthn.enroll
  - Challenges: challenge, webauthn.challenge
  - Verification: verify, webauthn.verify, challengeAndVerify
  - Management: unenroll, listFactors, getAuthenticatorAssuranceLevel
  - WebAuthn: authenticate, register

advanced-auth.md                   # Enterprise auth methods [0/6]
  - signInWithSSO, signInWithIdToken, signInWithWeb3, signInAnonymously
  - Auto-refresh: startAutoRefresh, stopAutoRefresh

# LOW Priority Files [~30 methods] - Admin & Specialized
admin-api.md                      # Server-side admin operations [0/14]
  - User CRUD: createUser, listUsers, getUserById, updateUserById, deleteUser
  - Invites: inviteUserByEmail, generateLink
  - Admin signOut, MFA management

oauth-server.md                   # OAuth 2.1 server (brief) [0/5]
  - getAuthorizationDetails, approveAuthorization, denyAuthorization
  - listGrants, revokeGrant

oauth-admin.md                    # OAuth client management (brief) [0/6]
  - listClients, createClient, getClient, updateClient, deleteClient
  - regenerateClientSecret

utilities.md                      # JWT verification, helpers [0/2]
  - getClaims, isThrowOnErrorEnabled

# Reference
implementation-matrix.md          # Implementation complexity analysis
```

---

## Priority Breakdown

### HIGH Priority (13 methods) - Core 80% Usage

**Session Management (5):**

- `getSession()` - Retrieve current session, auto-refresh if expired
- `setSession()` - Manually set/validate session (access_token, refresh_token)
- `refreshSession()` - Explicitly refresh session
- `getUser()` - Get authenticated user details (network request)
- `initialize()` - Initialize client, detect session from URL

**Sign In Methods (4):**

- `signInWithPassword()` - Email/phone + password authentication
- `signInWithOAuth()` - OAuth provider login (Google, GitHub, etc.)
- `signInWithOtp()` - Passwordless magic link/OTP login
- `exchangeCodeForSession()` - PKCE auth code exchange

**Sign Up (1):**

- `signUp()` - Register new user (email/phone + password)

**Sign Out (1):**

- `signOut()` - Sign out (scope: global/local/others)

**Verification (2):**

- `verifyOtp()` - Verify OTP/token from email/SMS
- `reauthenticate()` - Trigger reauthentication

### MEDIUM Priority (~25 methods) - Advanced 15% Usage

**User Management (5):**

- `updateUser()` - Update user metadata/email/password
- `resetPasswordForEmail()` - Send password reset link
- `resend()` - Resend confirmation/OTP
- `getUserIdentities()` - List linked identities
- `onAuthStateChange()` - Subscribe to auth events (SIGNED_IN, SIGNED_OUT, etc.)

**Identity Linking (2):**

- `linkIdentity()` - Link OAuth/OIDC identity to account
- `unlinkIdentity()` - Remove linked identity

**Multi-Factor Authentication (12):**

- Enrollment: `mfa.enroll()`, `mfa.webauthn.enroll()`
- Challenges: `mfa.challenge()`, `mfa.webauthn.challenge()`
- Verification: `mfa.verify()`, `mfa.webauthn.verify()`, `mfa.challengeAndVerify()`
- Management: `mfa.unenroll()`, `mfa.listFactors()`, `mfa.getAuthenticatorAssuranceLevel()`
- WebAuthn: `mfa.webauthn.authenticate()`, `mfa.webauthn.register()`

**Advanced Auth (6):**

- `signInWithSSO()` - Enterprise SSO (SAML)
- `signInWithIdToken()` - OIDC ID token authentication
- `signInWithWeb3()` - Web3 wallet authentication
- `signInAnonymously()` - Anonymous user creation
- `startAutoRefresh()` - Background token refresh
- `stopAutoRefresh()` - Stop background refresh

### LOW Priority (~30 methods) - Admin & Specialized 5% Usage

**Admin API (14):**

- User CRUD: `admin.createUser()`, `admin.listUsers()`, `admin.getUserById()`, `admin.updateUserById()`, `admin.deleteUser()`
- Invites: `admin.inviteUserByEmail()`, `admin.generateLink()`
- Admin: `admin.signOut()`, `admin.mfa.listFactors()`, `admin.mfa.deleteFactor()`

**OAuth 2.1 Server (5):**

- Authorization: `oauth.getAuthorizationDetails()`, `oauth.approveAuthorization()`, `oauth.denyAuthorization()`
- Grants: `oauth.listGrants()`, `oauth.revokeGrant()`

**OAuth Admin (6):**

- Client Management: `oauth.listClients()`, `oauth.createClient()`, `oauth.getClient()`, `oauth.updateClient()`, `oauth.deleteClient()`, `oauth.regenerateClientSecret()`

**Utilities (2):**

- `getClaims()` - Verify JWT offline (asymmetric keys)
- `isThrowOnErrorEnabled()` - Check error mode

---

## Documentation Format

### HIGH/MEDIUM Methods (Full Detail)

````markdown
#### `methodName(params)`

**Priority:** High/Medium | **Complexity:** Simple / Moderate / Complex

**Signature:**

```ts
method<Generics>(param: Type): Promise<ReturnType>
```
````

**Purpose:** One-line what it does

**Parameters:**

- `param` (Type) - Description, constraints, defaults

**Returns:**

```ts
{ data: DataType | null, error: AuthError | null }
```

**Usage:**

```ts
// Basic usage
const { data, error } = await client.auth.signInWithPassword({
   email: "user@example.com",
   password: "password123",
});

// With options
const { data, error } = await client.auth.signInWithPassword({
   email: "user@example.com",
   password: "password123",
   options: { captchaToken: "..." },
});
```

**Implementation Considerations:**

- **Backend Requirements:** JWT signing, token storage, email delivery, etc.
- **Security:** Password hashing (bcrypt), rate limiting, CSRF protection
- **Complexity:** Simple (basic CRUD) / Moderate (OAuth flow) / Complex (MFA, SAML)
- **Dependencies:** Email service, SMS provider, OAuth provider SDKs
- **Storage:** What needs to be persisted (users table, sessions, tokens)

**Authentication Flow:**

```
[Step-by-step flow diagram]
User submits credentials → Backend validates → Generate JWT → Store session → Return tokens
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /token?grant_type=password`
- **Request Body:** `{ email, password, gotrue_meta_security: { captcha_token } }`
- **Response:** `{ access_token, refresh_token, expires_in, user }`
- **Headers:** Content-Type, X-Client-Info

**Error Cases:**

- Invalid credentials: `invalid_credentials`
- User not confirmed: `email_not_confirmed`
- Too many attempts: `over_request_rate_limit`

**Related:** Links to similar methods

````

### LOW Priority Methods (Brief)

```markdown
#### `methodName(params)`
**Priority:** Low | **Complexity:** Simple/Moderate/Complex

**Signature:** `method(param: Type): Promise<ReturnType>`
**Purpose:** One-line description
**Usage:** `const { data, error } = await client.admin.methodName(param)`
**Implementation:** Brief note on complexity (e.g., "Requires admin JWT, service role key")
**Use Case:** When you would use this (e.g., "Enterprise apps with custom OAuth clients")
````

---

## Implementation Complexity Analysis

Instead of SQLite compatibility, analyze implementation complexity:

### ✅ Simple (Direct Implementation)

Methods with straightforward implementation:

- Password auth (with bcrypt)
- JWT generation/verification
- Basic user CRUD
- Session management

**Implementation Effort:** Low - Standard patterns

### ⚠️ Moderate (Requires Integration)

Methods requiring external services/SDKs:

- OAuth providers (Google, GitHub, etc.)
- Email sending (magic links, confirmations)
- SMS sending (OTP)
- Rate limiting
- PKCE flow

**Implementation Effort:** Medium - Integration + state management

### ❌ Complex (Advanced Features)

Methods requiring significant infrastructure:

- SAML/SSO (IdP integration)
- Multi-factor authentication (TOTP generation, WebAuthn ceremony)
- Web3 wallet verification (signature validation)
- OAuth 2.1 authorization server
- Admin API with service role validation

**Implementation Effort:** High - Specialized knowledge + infrastructure

---

## Key Authentication Flows

### 1. Password Authentication

```
signUp({ email, password })
  → Backend: Hash password, store user
  → [Optional] Email confirmation required
  → signInWithPassword({ email, password })
  → Backend: Verify password, generate JWT
  → getSession() → { access_token, refresh_token, user }
```

### 2. Passwordless/Magic Link

```
signInWithOtp({ email })
  → Backend: Generate OTP token, send email
  → User clicks link → initialize() detects ?token_hash=...
  → Backend: Verify token, create session
  → Session established
```

### 3. OAuth

```
signInWithOAuth({ provider: 'google' })
  → Redirect to Google OAuth consent
  → Google redirects back with code
  → initialize() detects ?code=...
  → Backend exchanges code for user info
  → Creates/links user account
  → Session established
```

### 4. PKCE Flow

```
signInWithPassword/signUp with PKCE enabled
  → Backend: Returns auth code instead of tokens
  → exchangeCodeForSession(code)
  → Backend: Validates code + code_verifier
  → Returns tokens
```

### 5. MFA Flow

```
User enrolls:
  mfa.enroll({ factorType: 'totp' })
    → Backend: Generate TOTP secret, return QR code
    → mfa.verify({ code })
    → Backend: Verify code, mark factor as verified

User authenticates (already signed in):
  mfa.challenge({ factorId })
    → Backend: Create challenge
    → mfa.verify({ challengeId, code })
    → Backend: Verify code, increase AAL to aal2
```

---

## Workflow

### Phase 1: Setup Structure

1. Create `/Users/dennis/Projects/docs/sdk/auth/` directory
2. Create progress.md with checklist
3. Create skeleton files

### Phase 2: Extract Source Info

For each method:

1. Read GoTrueClient.ts, GoTrueAdminApi.ts, webauthn.ts
2. Extract method signature, JSDoc, types
3. Identify authentication flow
4. Trace GoTrue backend endpoint mapping
5. Document request/response format

### Phase 3: Document HIGH Priority (13 methods)

**File:** core-authentication.md

Per method:

1. Copy JSDoc → Purpose
2. Extract TypeScript sig → Signature
3. Find usage examples
4. Map to GoTrue backend endpoint
5. Analyze implementation complexity
6. Document security considerations
7. Diagram authentication flow
8. Add error cases

### Phase 4: Document MEDIUM Priority (~25 methods)

**Files:** user-management.md, identity-linking.md, mfa.md, advanced-auth.md

Same as Phase 3, moderate detail level

### Phase 5: Document LOW Priority (~30 methods)

**Files:** admin-api.md, oauth-server.md, oauth-admin.md, utilities.md

Brief format only

### Phase 6: Create References

1. api-overview.md - Summary of all methods
2. implementation-matrix.md - Complexity analysis by method

---

## Critical Source Files

**Implementation reference:**

- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/GoTrueClient.ts` - Main client (3865 lines)
- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/GoTrueAdminApi.ts` - Admin operations (551 lines)
- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/lib/webauthn.ts` - WebAuthn API
- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/lib/types.ts` - Type definitions
- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/lib/errors.ts` - Error types

**Test references:**

- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/test/` - Usage examples

---

## Security Considerations

Each method should document:

- **Authentication Required:** User JWT, Admin JWT, or Public
- **Rate Limiting:** Should this be rate limited?
- **CSRF Protection:** Required for state-changing operations
- **Input Validation:** Email format, password strength, phone number format
- **Output Sanitization:** What user data is returned?
- **Audit Logging:** Should this action be logged?

---

## Progress Tracking

Update after completing each file:

**Completed:** 18/70 methods (25.7%)

- [x] api-overview.md (reference)
- [x] core-authentication.md (13 methods)
- [x] user-management.md (5 methods)
- [ ] identity-linking.md (2 methods)
- [ ] mfa.md (12 methods)
- [ ] advanced-auth.md (6 methods)
- [ ] admin-api.md (14 methods)
- [ ] oauth-server.md (5 methods, brief)
- [ ] oauth-admin.md (6 methods, brief)
- [ ] utilities.md (2 methods, brief)
- [x] implementation-matrix.md (reference)

**Completion Criteria:**

- [ ] All ~70 methods documented (varying detail by priority)
- [ ] Implementation complexity analyzed for all
- [ ] Master complexity matrix complete
- [ ] Examples extracted from source
- [ ] Authentication flows diagrammed
- [ ] Security considerations documented
- [ ] GoTrue backend endpoints mapped

---

## Unresolved Questions

(answers provided after "->" of each question)

1. **Implementation Target:** Building GoTrue-compatible backend or custom auth system?
   -> Building GoTrue-compatible backend, but with only the features that are needed for the project. The backend will be in TypeScript. Scope is that we build a system that is compatible, but not mission critical.

2. **OAuth Scope:** Which OAuth providers priority? (Google, GitHub only or full list?)
   -> only the most important ones, like Google, GitHub, Apple, etc.

3. **MFA Priority:** Required for MVP or defer?
   -> defer to LOW priority

4. **Admin API:** Server-side only or expose some to client?
   -> server-side only

5. **WebAuthn:** Include experimental WebAuthn in HIGH priority or defer to LOW?
   -> defer to LOW priority

6. **Enterprise Features:** SSO/SAML truly LOW priority or MEDIUM if targeting enterprise?
   -> defer to LOW priority
