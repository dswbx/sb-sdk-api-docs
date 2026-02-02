# Implementation Complexity Matrix

Analysis of implementation complexity for all auth-js methods. Helps prioritize backend implementation based on difficulty.

**Focus:** Implementation effort, not SQLite compatibility (unlike Database APIs)

---

## Complexity Levels

### ✅ Simple - Direct Implementation

Straightforward backend logic, standard patterns, minimal external dependencies.

**Implementation Effort:** Low (1-2 days per method)
**Requirements:** Basic HTTP server, JWT library, database
**Skills:** Backend fundamentals

### ⚠️ Moderate - Requires Integration

External service integration, state management, multi-step flows.

**Implementation Effort:** Medium (3-5 days per method)
**Requirements:** External APIs (email/SMS), OAuth SDKs, complex state
**Skills:** API integration, OAuth flows, async workflows

### ❌ Complex - Advanced Features

Specialized protocols, cryptographic operations, significant infrastructure.

**Implementation Effort:** High (1-2 weeks per method)
**Requirements:** Advanced protocols (SAML, WebAuthn), crypto libraries, specialized knowledge
**Skills:** Security protocols, cryptography, enterprise auth standards

---

## Complexity by Method

### Session Management (5 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `getSession()` | ✅ Simple | Read from storage, decode JWT | Storage adapter |
| `setSession()` | ✅ Simple | Validate JWT, store session | JWT library |
| `refreshSession()` | ✅ Simple | Exchange refresh token for new JWT | JWT generation, token storage |
| `getUser()` | ✅ Simple | Validate JWT, query user by ID | JWT validation, database |
| `initialize()` | ⚠️ Moderate | URL parsing, PKCE validation, storage recovery | PKCE crypto, URL parsing |

**Overall:** Mostly simple, initialize() requires PKCE understanding.

---

### Sign In (4 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `signInWithPassword()` | ✅ Simple | Password hash comparison, JWT generation | bcrypt, JWT library |
| `signInWithOAuth()` | ⚠️ Moderate | OAuth handshake, provider integration, PKCE | OAuth provider SDKs, PKCE |
| `signInWithOtp()` | ⚠️ Moderate | Email/SMS delivery, OTP generation, token storage | Email service, SMS provider (Twilio) |
| `exchangeCodeForSession()` | ⚠️ Moderate | PKCE verification, code validation | PKCE crypto, code storage |

**Overall:** Password auth simple, others need external services.

**OAuth Providers (Moderate each):**
- Google: OAuth 2.0 client setup
- GitHub: OAuth Apps registration
- Apple: Sign in with Apple (special requirements)
- Azure: Microsoft identity platform
- Others: 20+ providers supported

---

### Sign Up (1 method)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `signUp()` | ⚠️ Moderate | Password hashing, email/SMS confirmation, PKCE | bcrypt, email/SMS service |

**Overall:** Email confirmation adds moderate complexity.

---

### Sign Out (1 method)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `signOut()` | ✅ Simple | Revoke refresh tokens, track revocation | Token revocation table |

**Overall:** Simple database operation.

---

### Verification (2 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `verifyOtp()` | ✅ Simple | Token hash comparison, create session | Token storage, JWT generation |
| `reauthenticate()` | ⚠️ Moderate | OTP generation, email/SMS delivery | Email/SMS service |

**Overall:** Verification simple, delivery moderate.

---

### User Management (5 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `updateUser()` | ⚠️ Moderate | Email/phone change confirmation, password hashing | Email/SMS service, bcrypt |
| `resetPasswordForEmail()` | ⚠️ Moderate | Token generation, email delivery, PKCE | Email service |
| `resend()` | ⚠️ Moderate | Lookup context, resend email/SMS | Email/SMS service |
| `getUserIdentities()` | ✅ Simple | Query database, return identities | Database |
| `onAuthStateChange()` | ✅ Simple | Client-side event emitter | None (client-only) |

**Overall:** Email/SMS operations moderate, queries simple.

---

### Identity Linking (2 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `linkIdentity()` | ⚠️ Moderate | OAuth flow, identity merging, conflict resolution | OAuth providers |
| `unlinkIdentity()` | ✅ Simple | Delete identity record | Database |

**Overall:** Linking moderate (OAuth), unlinking simple.

---

### Multi-Factor Authentication (12 methods)

#### Core MFA (7 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `mfa.enroll()` | ⚠️ Moderate | TOTP secret generation, QR code, SMS delivery | TOTP library (otplib), QR library, SMS |
| `mfa.challenge()` | ✅ Simple | Create challenge record, send SMS (phone) | SMS (phone only) |
| `mfa.verify()` | ⚠️ Moderate | TOTP/SMS verification, AAL elevation, session update | TOTP validation, JWT |
| `mfa.unenroll()` | ✅ Simple | Delete factor, check AAL | Database |
| `mfa.listFactors()` | ✅ Simple | Query user factors | Database |
| `mfa.challengeAndVerify()` | ⚠️ Moderate | Combined challenge + verify | Same as challenge + verify |
| `mfa.getAuthenticatorAssuranceLevel()` | ✅ Simple | Parse JWT AAL claim | JWT parsing |

**TOTP Implementation:**
- Secret generation (crypto random)
- Base32 encoding
- QR code generation
- TOTP algorithm (RFC 6238)
- Time window validation

**Phone MFA:**
- SMS delivery (Twilio)
- OTP generation
- Rate limiting

#### WebAuthn (5 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `mfa.webauthn.enroll()` | ❌ Complex | WebAuthn registration ceremony, credential storage | WebAuthn library (@simplewebauthn/server) |
| `mfa.webauthn.challenge()` | ❌ Complex | Challenge generation, browser API coordination | WebAuthn crypto |
| `mfa.webauthn.verify()` | ❌ Complex | Attestation/assertion verification, signature validation | WebAuthn verification |
| `mfa.webauthn.authenticate()` | ❌ Complex | Full authentication ceremony | Same as challenge + verify |
| `mfa.webauthn.register()` | ❌ Complex | Full registration ceremony | Same as enroll + challenge + verify |

**WebAuthn Requirements:**
- **Protocol:** W3C WebAuthn API
- **Crypto:** Public key cryptography, challenge-response
- **Storage:** Credential IDs, public keys, counters
- **Browser:** navigator.credentials API
- **Server:** Attestation validation, assertion verification
- **Libraries:** @simplewebauthn/server, @simplewebauthn/browser
- **Expertise:** PKI, FIDO2, CTAP2 protocols

**Overall:** TOTP/SMS moderate, WebAuthn complex (specialized knowledge).

---

### Advanced Authentication (6 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `signInWithSSO()` | ❌ Complex | SAML 2.0 or OIDC, IdP integration, metadata exchange | SAML library (samlify), OIDC provider |
| `signInWithIdToken()` | ⚠️ Moderate | OIDC token validation, nonce verification | OIDC validation |
| `signInWithWeb3()` | ❌ Complex | SIWE/SIWS message verification, signature validation | ethers.js, @solana/web3.js |
| `signInAnonymously()` | ✅ Simple | Create user without credentials | Database, JWT |
| `startAutoRefresh()` | ✅ Simple | Timer management, refresh logic | Client-side timer |
| `stopAutoRefresh()` | ✅ Simple | Cancel timer | Client-side timer |

**SSO/SAML Complexity:**
- **SAML 2.0:** XML signing, metadata exchange, IdP configuration
- **OIDC:** Similar to OAuth but with additional ID token validation
- **Enterprise:** Multi-tenant, SSO domains, provider discovery

**Web3 Complexity:**
- **SIWE (Ethereum):** EIP-4361 message format, signature recovery
- **SIWS (Solana):** Solana-specific signature verification
- **Wallet Integration:** window.ethereum, wallet connect
- **Signature Validation:** Elliptic curve cryptography

**Overall:** SSO and Web3 very complex, others simple/moderate.

---

### Admin API (14 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `admin.createUser()` | ✅ Simple | Create user, skip confirmation | Database, JWT |
| `admin.listUsers()` | ✅ Simple | Query users with pagination | Database |
| `admin.getUserById()` | ✅ Simple | Query user by ID | Database |
| `admin.updateUserById()` | ✅ Simple | Update user record | Database |
| `admin.deleteUser()` | ✅ Simple | Delete user + cascades | Database |
| `admin.inviteUserByEmail()` | ⚠️ Moderate | Generate invite, send email | Email service |
| `admin.generateLink()` | ✅ Simple | Create magic link token | Token generation |
| `admin.signOut()` | ✅ Simple | Revoke user tokens | Token revocation |
| `admin.mfa.listFactors()` | ✅ Simple | Query user factors | Database |
| `admin.mfa.deleteFactor()` | ✅ Simple | Delete factor | Database |

**Service Role Key:**
- Special JWT with elevated permissions
- Required for all admin operations
- Should be stored securely server-side

**Overall:** Mostly simple CRUD, invite requires email.

---

### OAuth 2.1 Server (5 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `oauth.getAuthorizationDetails()` | ⚠️ Moderate | Parse authorization request, validate client | OAuth 2.1 spec |
| `oauth.approveAuthorization()` | ⚠️ Moderate | Generate authorization code, PKCE | PKCE, code generation |
| `oauth.denyAuthorization()` | ✅ Simple | Return error response | None |
| `oauth.listGrants()` | ✅ Simple | Query user grants | Database |
| `oauth.revokeGrant()` | ✅ Simple | Revoke grant | Database |

**OAuth 2.1 Server Requirements:**
- Authorization endpoint
- Token endpoint (already exists)
- Client management (see OAuth Admin)
- PKCE support
- Scope management
- Consent screen UI

**Overall:** Moderate - implementing OAuth authorization server.

---

### OAuth Admin (6 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `oauth.listClients()` | ✅ Simple | Query OAuth clients | Database |
| `oauth.createClient()` | ✅ Simple | Create client record | Database |
| `oauth.getClient()` | ✅ Simple | Query client by ID | Database |
| `oauth.updateClient()` | ✅ Simple | Update client | Database |
| `oauth.deleteClient()` | ✅ Simple | Delete client | Database |
| `oauth.regenerateClientSecret()` | ✅ Simple | Generate new secret | Crypto random |

**Overall:** Simple CRUD operations.

---

### Utilities (2 methods)

| Method | Complexity | Reason | Dependencies |
|--------|------------|--------|--------------|
| `getClaims()` | ⚠️ Moderate | Fetch JWKS, verify JWT signature, parse claims | JWKS fetch, JWT verify |
| `isThrowOnErrorEnabled()` | ✅ Simple | Return config value | None |

**JWKS (JSON Web Key Set):**
- Fetch public keys from `/.well-known/jwks.json`
- Cache keys with TTL
- Verify JWT signature asymmetrically
- Parse and return claims

**Overall:** JWKS verification moderate, config simple.

---

## Summary by Complexity

### ✅ Simple (30 methods) - 43%

Direct implementation, minimal dependencies.

**Session:** getSession, setSession, refreshSession, getUser
**Sign Out:** signOut
**Verification:** verifyOtp
**User Management:** getUserIdentities, onAuthStateChange
**Identity:** unlinkIdentity
**MFA:** mfa.challenge, mfa.unenroll, mfa.listFactors, mfa.getAuthenticatorAssuranceLevel
**Advanced:** signInAnonymously, startAutoRefresh, stopAutoRefresh
**Admin:** createUser, listUsers, getUserById, updateUserById, deleteUser, generateLink, signOut, mfa.listFactors, mfa.deleteFactor
**OAuth Server:** denyAuthorization, listGrants, revokeGrant
**OAuth Admin:** All 6 methods
**Utilities:** isThrowOnErrorEnabled

### ⚠️ Moderate (30 methods) - 43%

External service integration, multi-step flows.

**Session:** initialize
**Sign In:** signInWithOAuth, signInWithOtp, exchangeCodeForSession
**Sign Up:** signUp
**Verification:** reauthenticate
**User Management:** updateUser, resetPasswordForEmail, resend
**Identity:** linkIdentity
**MFA:** mfa.enroll, mfa.verify, mfa.challengeAndVerify
**Advanced:** signInWithIdToken
**Admin:** admin.inviteUserByEmail
**OAuth Server:** getAuthorizationDetails, approveAuthorization
**Utilities:** getClaims

**Plus OAuth providers (each moderate):**
- Google, GitHub, Apple, Azure, Facebook, Twitter, GitLab, Bitbucket, Discord, Slack, Spotify, Twitch, LinkedIn, Notion, etc.

### ❌ Complex (10 methods) - 14%

Advanced protocols, specialized knowledge required.

**MFA WebAuthn:** All 5 methods (enroll, challenge, verify, authenticate, register)
**Advanced:** signInWithSSO (SAML/OIDC), signInWithWeb3 (SIWE/SIWS)

---

## Implementation Roadmap

### Phase 1: Core Auth (Simple) - Week 1-2

Focus on essential authentication without external dependencies.

**Methods (13):**
- Session: getSession, setSession, refreshSession, getUser
- Sign In: signInWithPassword
- Sign Up: (without email confirmation)
- Sign Out: signOut
- Verification: verifyOtp (basic)
- User Management: getUserIdentities, onAuthStateChange
- Advanced: signInAnonymously

**Backend Requirements:**
- HTTP server (Express/Fastify)
- PostgreSQL database
- JWT library (jsonwebtoken)
- bcrypt for passwords
- Basic token storage

**Estimated Effort:** 2 weeks

### Phase 2: Email Integration (Moderate) - Week 3-4

Add email-based flows.

**Methods (6):**
- Session: initialize (email callbacks)
- Sign In: signInWithOtp (email only)
- Sign Up: signUp (with email confirmation)
- User Management: updateUser (email changes), resetPasswordForEmail, resend (email)
- Verification: reauthenticate (email)

**Backend Requirements:**
- Email service (SMTP/SendGrid/Postmark)
- Email templates
- Token generation
- PKCE implementation

**Estimated Effort:** 2 weeks

### Phase 3: OAuth (Moderate) - Week 5-7

Implement OAuth providers.

**Methods (2 + providers):**
- Sign In: signInWithOAuth, exchangeCodeForSession
- Identity: linkIdentity, unlinkIdentity

**Backend Requirements:**
- OAuth 2.0 client library
- Provider apps (Google, GitHub, Apple)
- PKCE support
- State management

**Per Provider:** 2-3 days setup + testing

**Estimated Effort:** 3 weeks (3-5 providers)

### Phase 4: SMS & MFA (Moderate to Complex) - Week 8-10

Add SMS and basic MFA.

**Methods (5):**
- Sign In: signInWithOtp (SMS), exchangeCodeForSession (SMS)
- MFA: mfa.enroll (TOTP/phone), mfa.challenge, mfa.verify, mfa.unenroll, mfa.listFactors, mfa.challengeAndVerify, mfa.getAuthenticatorAssuranceLevel

**Backend Requirements:**
- SMS provider (Twilio)
- TOTP library (otplib)
- QR code generation
- AAL tracking

**Estimated Effort:** 3 weeks

### Phase 5: Admin & Utilities (Simple) - Week 11

Server-side admin operations.

**Methods (12):**
- Admin: All methods (createUser, listUsers, etc.)
- Utilities: getClaims, isThrowOnErrorEnabled

**Backend Requirements:**
- Service role validation
- JWKS endpoint
- Admin middleware

**Estimated Effort:** 1 week

### Phase 6: Advanced (Complex) - Optional

Enterprise features, defer or skip for MVP.

**Methods (7):**
- Advanced: signInWithSSO, signInWithWeb3
- MFA WebAuthn: All 5 methods
- OAuth Server: All 5 methods

**Backend Requirements:**
- SAML library (samlify)
- Web3 libraries (ethers.js, @solana/web3.js)
- WebAuthn library (@simplewebauthn/server)
- OAuth 2.1 authorization server

**Estimated Effort:** 4-6 weeks (if needed)

---

## Dependencies Summary

### Required Libraries

**Core:**
- `jsonwebtoken` - JWT generation/validation
- `bcrypt` - Password hashing
- `uuid` - Unique IDs

**Email/SMS:**
- Email: `nodemailer`, SendGrid SDK, or Postmark SDK
- SMS: Twilio SDK

**OAuth:**
- `passport` (optional) or manual OAuth 2.0
- Provider-specific SDKs (optional)

**MFA:**
- `otplib` - TOTP generation
- `qrcode` - QR code generation
- `@simplewebauthn/server` - WebAuthn (complex)

**Advanced:**
- `samlify` - SAML 2.0 (SSO)
- `ethers` or `@solana/web3.js` - Web3 wallets

### External Services

**Required:**
- Database (PostgreSQL recommended)
- Email provider (SMTP, SendGrid, Postmark, etc.)

**Optional:**
- SMS provider (Twilio for phone/WhatsApp)
- OAuth provider apps (Google, GitHub, Apple, etc.)
- SAML IdP (for SSO)

### Infrastructure

**Development:**
- Node.js 18+
- PostgreSQL 14+
- Redis (optional, for rate limiting)

**Production:**
- Load balancer with SSL
- Database with backups
- Email/SMS quotas
- Monitoring (errors, rate limits)

---

## Security Implementation Notes

### Password Hashing
- Use bcrypt with cost factor 10-12
- Never log or return passwords
- Implement rate limiting (5 attempts per 15 min)

### JWT Tokens
- Access token: 1 hour expiry (configurable)
- Refresh token: 30 days expiry (configurable)
- Use asymmetric keys (RS256) for production
- Rotate signing keys periodically

### PKCE
- Use S256 (SHA-256) challenge method
- Verify code_verifier before issuing tokens
- Single-use authorization codes (expire in 10 min)

### Rate Limiting
- Sign in: 5 attempts per 15 min per IP
- Sign up: 3 accounts per hour per IP
- Email: 5 emails per hour per user
- SMS: 3 SMS per hour per user
- OTP verification: 5 attempts per OTP

### Token Storage
- Store refresh tokens hashed (SHA-256)
- Index by token hash for revocation lookups
- Cascade delete on user deletion
- Track refresh token families (detect theft)

### Email/Phone Verification
- Use cryptographically secure random tokens
- Hash tokens before storage
- Expire after 1 hour (configurable)
- Single-use (mark as used after verification)

---

## Testing Strategy

### Unit Tests
- JWT generation/validation
- Password hashing/comparison
- TOTP generation/validation
- PKCE challenge/verifier

### Integration Tests
- Full auth flows end-to-end
- Email/SMS delivery (mocked)
- OAuth flows (mocked providers)
- Token refresh flows

### Security Tests
- Rate limiting enforcement
- Token expiration
- PKCE attack prevention
- Session hijacking prevention

---

## Documentation Status

**Documented Methods:** 18/70 (25.7%)

- ✅ HIGH Priority: 13/13 complete
- 🔄 MEDIUM Priority: 5/25 (user-management.md complete)
- ⏳ LOW Priority: 0/30+ planned

See [api-overview.md](./api-overview.md) and [progress.md](./progress.md) for details.
