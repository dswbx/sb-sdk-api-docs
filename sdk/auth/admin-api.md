# Admin API

**Priority:** LOW
**Methods:** 10
**Status:** Complete

Server-side admin operations for user management, invites, and MFA administration. All methods require service role key (admin JWT). Never expose `service_role` key in the browser.

---

## User CRUD

#### `admin.createUser(attributes)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `createUser(attributes: AdminUserAttributes): Promise<UserResponse>`
**Purpose:** Create a new user with specified attributes (email, phone, password, metadata).
**Usage:**
```ts
const { data, error } = await supabase.auth.admin.createUser({
  email: 'user@example.com',
  password: 'securePassword123',
  email_confirm: true,           // skip email confirmation
  user_metadata: { name: 'John' },
})
```
**Implementation:** POST `/admin/users`. AdminUserAttributes supports: `email`, `phone`, `password`, `email_confirm`, `phone_confirm`, `user_metadata`, `app_metadata`, `ban_duration`, `role`.
**Use Case:** Server-side user provisioning, admin dashboards, migration scripts.

---

#### `admin.listUsers(params?)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `listUsers(params?: PageParams): Promise<{ data: { users: User[]; aud: string } & Pagination; error: null } | { data: { users: [] }; error: AuthError }>`
**Purpose:** List all users with optional pagination.
**Usage:**
```ts
const { data, error } = await supabase.auth.admin.listUsers({ page: 1, perPage: 50 })
// data: { users, aud, nextPage, lastPage, total }
```
**Implementation:** GET `/admin/users`. Pagination via `page`/`per_page` query params. Total count from `x-total-count` header, page links from `link` header.
**Use Case:** Admin dashboards, user management panels, bulk operations.

---

#### `admin.getUserById(uid)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `getUserById(uid: string): Promise<UserResponse>`
**Purpose:** Get a single user by UUID.
**Usage:** `const { data, error } = await supabase.auth.admin.getUserById('uuid')`
**Implementation:** GET `/admin/users/{uid}`. Validates UUID format before request.
**Use Case:** User detail pages, admin lookup, support tools.

---

#### `admin.updateUserById(uid, attributes)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `updateUserById(uid: string, attributes: AdminUserAttributes): Promise<UserResponse>`
**Purpose:** Update user data directly without confirmation flows.
**Usage:**
```ts
const { data, error } = await supabase.auth.admin.updateUserById('uuid', {
  email: 'new@example.com',
  email_confirm: true,
  user_metadata: { role: 'admin' },
  ban_duration: '24h',           // ban user for 24 hours
})
```
**Implementation:** PUT `/admin/users/{uid}`. Changes applied immediately — no email/phone confirmation required. Can update `email`, `phone`, `password`, `email_confirm`, `phone_confirm`, `user_metadata`, `app_metadata`, `ban_duration`, `role`.
**Use Case:** Admin-initiated email changes, banning users, updating app_metadata for RBAC.

---

#### `admin.deleteUser(id, shouldSoftDelete?)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `deleteUser(id: string, shouldSoftDelete?: boolean): Promise<UserResponse>`
**Purpose:** Delete a user permanently or soft-delete.
**Usage:**
```ts
// Hard delete (default)
const { data, error } = await supabase.auth.admin.deleteUser('uuid')

// Soft delete (preserves hashed user ID)
const { data, error } = await supabase.auth.admin.deleteUser('uuid', true)
```
**Implementation:** DELETE `/admin/users/{id}` with body `{ should_soft_delete }`. Soft deletion allows identification from hashed ID but is not reversible. Defaults to `false` (hard delete).
**Use Case:** Account deletion requests, GDPR compliance, user cleanup.

---

## Invites & Links

#### `admin.inviteUserByEmail(email, options?)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `inviteUserByEmail(email: string, options?: { data?: object; redirectTo?: string }): Promise<UserResponse>`
**Purpose:** Send an invite link to an email address.
**Usage:**
```ts
const { data, error } = await supabase.auth.admin.inviteUserByEmail(
  'user@example.com',
  {
    data: { role: 'editor' },
    redirectTo: 'https://myapp.com/welcome',
  }
)
```
**Implementation:** POST `/invite`. `data` maps to `auth.users.user_metadata`. `redirectTo` appended to the email link.
**Use Case:** Team invitation flows, onboarding new users via email.

---

#### `admin.generateLink(params)`
**Priority:** Low | **Complexity:** Moderate

**Signature:** `generateLink(params: GenerateLinkParams): Promise<GenerateLinkResponse>`
**Purpose:** Generate email links and OTPs for custom email delivery (bypass built-in email service).
**Usage:**
```ts
// Generate signup confirmation link
const { data, error } = await supabase.auth.admin.generateLink({
  type: 'signup',
  email: 'user@example.com',
  password: 'securePassword123',
  options: { redirectTo: 'https://myapp.com/confirm' },
})
// data.properties: { action_link, hashed_token, email_otp, redirect_to, verification_type }

// Generate magic link
const { data, error } = await supabase.auth.admin.generateLink({
  type: 'magiclink',
  email: 'user@example.com',
})

// Generate email change link
const { data, error } = await supabase.auth.admin.generateLink({
  type: 'email_change_current',
  email: 'user@example.com',
  newEmail: 'new@example.com',
})
```
**Implementation:** POST `/admin/generate_link`. Types: `signup`, `invite`, `magiclink`, `recovery`, `email_change_current`, `email_change_new`. Returns `action_link` and `hashed_token` for custom email templates.
**Use Case:** Custom email providers (SendGrid, Postmark), custom email templates, headless auth flows.

---

## Admin Operations

#### `admin.signOut(jwt, scope?)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `signOut(jwt: string, scope?: SignOutScope): Promise<{ data: null; error: AuthError | null }>`
**Purpose:** Revoke a user's session server-side.
**Usage:**
```ts
// Sign out specific session
const { error } = await supabase.auth.admin.signOut(userJwt)

// Sign out all sessions (global)
const { error } = await supabase.auth.admin.signOut(userJwt, 'global')
```
**Implementation:** POST `/logout?scope={scope}`. Scope options: `global` (all sessions), `local` (current session), `others` (all except current). Defaults to `global`.
**Use Case:** Admin-initiated session revocation, security incident response, force logout.

---

#### `admin.mfa.listFactors(params)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `mfa.listFactors(params: { userId: string }): Promise<AuthMFAAdminListFactorsResponse>`
**Purpose:** List all MFA factors enrolled by a specific user.
**Usage:**
```ts
const { data, error } = await supabase.auth.admin.mfa.listFactors({
  userId: 'uuid',
})
// data: { factors: Factor[] }
```
**Implementation:** GET `/admin/users/{userId}/factors`. Returns array of enrolled factors (TOTP, WebAuthn, etc.).
**Use Case:** Admin MFA management, support tools for MFA troubleshooting.

---

#### `admin.mfa.deleteFactor(params)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `mfa.deleteFactor(params: { userId: string; id: string }): Promise<AuthMFAAdminDeleteFactorResponse>`
**Purpose:** Delete a specific MFA factor for a user.
**Usage:**
```ts
const { error } = await supabase.auth.admin.mfa.deleteFactor({
  userId: 'user-uuid',
  id: 'factor-uuid',
})
```
**Implementation:** DELETE `/admin/users/{userId}/factors/{id}`. Validates both UUIDs. Removes factor permanently.
**Use Case:** Helping users locked out of MFA, admin recovery flows, factor cleanup.

---

## Summary

| Method | HTTP | Endpoint | Complexity |
|--------|------|----------|-----------|
| `createUser` | POST | `/admin/users` | Simple |
| `listUsers` | GET | `/admin/users` | Simple |
| `getUserById` | GET | `/admin/users/{uid}` | Simple |
| `updateUserById` | PUT | `/admin/users/{uid}` | Simple |
| `deleteUser` | DELETE | `/admin/users/{uid}` | Simple |
| `inviteUserByEmail` | POST | `/invite` | Simple |
| `generateLink` | POST | `/admin/generate_link` | Moderate |
| `signOut` | POST | `/logout` | Simple |
| `mfa.listFactors` | GET | `/admin/users/{uid}/factors` | Simple |
| `mfa.deleteFactor` | DELETE | `/admin/users/{uid}/factors/{id}` | Simple |

**Key notes:**
- All methods require `service_role` key — server-side only
- `updateUserById` bypasses confirmation flows (changes applied immediately)
- `deleteUser` supports soft-delete for GDPR compliance
- `generateLink` enables custom email delivery workflows
- OAuth client admin methods documented separately in `oauth-admin.md`
