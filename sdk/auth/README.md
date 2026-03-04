# Auth API Documentation

Documentation for Supabase auth-js (GoTrue) APIs. Focus on understanding implementation patterns for building custom authentication backend.

**Status:** Complete
**Total Methods:** 68
**Completed:** 68/68 (100%)

---

## Quick Links

- [Progress Tracking](./progress.md)
- [API Overview](./api-overview.md) - Summary & quick reference ✅
- [Implementation Matrix](./implementation-matrix.md) - Complexity analysis ✅

---

## Documentation Files

### HIGH Priority (13 methods) - Core 80% Usage

**[Core Authentication](./core-authentication.md)** [13/13] ✅
- Session: getSession, setSession, refreshSession, getUser, initialize
- Sign In: signInWithPassword, signInWithOAuth, signInWithOtp, exchangeCodeForSession
- Sign Up: signUp
- Sign Out: signOut
- Verification: verifyOtp, reauthenticate

---

### MEDIUM Priority (25 methods) - Advanced 15% Usage

**[User Management](./user-management.md)** [5/5] ✅
- updateUser, resetPasswordForEmail, resend, getUserIdentities, onAuthStateChange

**[Identity Linking](./identity-linking.md)** [2/2] ✅
- linkIdentity (OAuth redirect + ID token overloads), unlinkIdentity

**[Multi-Factor Authentication](./mfa.md)** [12/12] ✅
- Enrollment: enroll, webauthn.enroll
- Challenges: challenge, webauthn.challenge
- Verification: verify, webauthn.verify, challengeAndVerify
- Management: unenroll, listFactors, getAuthenticatorAssuranceLevel
- WebAuthn: authenticate, register

**[Advanced Auth](./advanced-auth.md)** [6/6] ✅
- signInWithSSO, signInWithIdToken, signInWithWeb3, signInAnonymously, startAutoRefresh, stopAutoRefresh

---

### LOW Priority (30 methods) - Admin & Specialized 5% Usage

**[Admin API](./admin-api.md)** [10/10] ✅
- User CRUD: createUser, listUsers, getUserById, updateUserById, deleteUser
- Invites: inviteUserByEmail, generateLink
- Operations: signOut, mfa.listFactors, mfa.deleteFactor

**[OAuth Server](./oauth-server.md)** [5/5] ✅
- Consent: getAuthorizationDetails, approveAuthorization, denyAuthorization
- Grants: listGrants, revokeGrant

**[OAuth Admin](./oauth-admin.md)** [6/6] ✅
- Client CRUD: listClients, createClient, getClient, updateClient, deleteClient, regenerateClientSecret

**[Utilities](./utilities.md)** [2/2] ✅
- getClaims (offline JWT verification via JWKS), isThrowOnErrorEnabled

---

## Documentation Format

### HIGH/MEDIUM Methods
Full detail including:
- Signature with TypeScript types
- Purpose, parameters, returns
- Usage examples (basic + advanced)
- Implementation considerations (backend requirements, security, complexity)
- Authentication flows (diagrams)
- GoTrue backend endpoint mapping
- Error cases
- Related methods

### LOW Methods
Brief format:
- One-line purpose
- Basic signature
- Single usage example
- Implementation note
- Use case

---

## Implementation Complexity

- **✅ Simple** - Direct implementation (password auth, JWT, session management)
- **⚠️ Moderate** - Requires integration (OAuth, email/SMS, rate limiting)
- **❌ Complex** - Advanced infrastructure (SAML/SSO, MFA, Web3, OAuth server)

See [Implementation Matrix](./implementation-matrix.md) for full analysis.

---

## Source References

- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/GoTrueClient.ts`
- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/GoTrueAdminApi.ts`
- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/lib/webauthn.ts`
- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/lib/types.ts`
- `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/lib/errors.ts`

---

## Key Decisions

From auth-api-plan.md:

- **Target:** GoTrue-compatible backend, TypeScript, non-mission-critical
- **OAuth:** Major providers only (Google, GitHub, Apple)
- **MFA:** Documented at MEDIUM priority (full detail)
- **Admin API:** Server-side only
- **WebAuthn:** Documented at MEDIUM priority (full detail within mfa.md)
- **SSO/SAML:** Documented at MEDIUM priority (in advanced-auth.md)
