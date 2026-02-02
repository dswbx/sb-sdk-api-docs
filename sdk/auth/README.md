# Auth API Documentation

Documentation for Supabase auth-js (GoTrue) APIs. Focus on understanding implementation patterns for building custom authentication backend.

**Status:** In Progress
**Total Methods:** ~70
**Completed:** 18/70 (25.7%)

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

**[Identity Linking](./identity-linking.md)** [0/2]
- linkIdentity, unlinkIdentity

**[Multi-Factor Authentication](./mfa.md)** [0/12]
- Enrollment, challenges, verification, management, WebAuthn

**[Advanced Auth](./advanced-auth.md)** [0/6]
- signInWithSSO, signInWithIdToken, signInWithWeb3, signInAnonymously, auto-refresh

---

### LOW Priority (30 methods) - Admin & Specialized 5% Usage

**[Admin API](./admin-api.md)** [0/14]
- User CRUD, invites, admin operations (brief format)

**[OAuth Server](./oauth-server.md)** [0/5]
- Authorization details, grants (brief format)

**[OAuth Admin](./oauth-admin.md)** [0/6]
- Client management (brief format)

**[Utilities](./utilities.md)** [0/2]
- JWT verification, helpers (brief format)

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

Instead of SQLite compatibility, analyze implementation complexity:

- **✅ Simple** - Direct implementation (password auth, JWT, session management)
- **⚠️ Moderate** - Requires integration (OAuth, email/SMS, rate limiting)
- **❌ Complex** - Advanced infrastructure (SAML/SSO, MFA, Web3, OAuth server)

See [Implementation Matrix](./implementation-matrix.md) for full analysis.

---

## Source References

Implementation source:
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
- **MFA:** LOW priority (deferred)
- **Admin API:** Server-side only
- **WebAuthn:** LOW priority (deferred)
- **SSO/SAML:** LOW priority (deferred)
