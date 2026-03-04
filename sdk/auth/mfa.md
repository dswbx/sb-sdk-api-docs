# Multi-Factor Authentication (MFA)

**Priority:** MEDIUM
**Methods:** 12
**Status:** Complete

Multi-factor authentication including TOTP, SMS, and WebAuthn.

---

## Enrollment

### `mfa.enroll(params)`

**Priority:** Medium | **Complexity:** Moderate

**Signature:**

```ts
// TOTP
async mfa.enroll(params: MFAEnrollTOTPParams): Promise<AuthMFAEnrollTOTPResponse>

// Phone
async mfa.enroll(params: MFAEnrollPhoneParams): Promise<AuthMFAEnrollPhoneResponse>

// WebAuthn
async mfa.enroll(params: MFAEnrollWebauthnParams): Promise<AuthMFAEnrollWebauthnResponse>

// Union
async mfa.enroll(params: MFAEnrollParams): Promise<AuthMFAEnrollResponse>

// Param types
type MFAEnrollParamsBase<T extends FactorType> = {
  factorType: T
  friendlyName?: string
}

type MFAEnrollTOTPParams = MFAEnrollParamsBase<'totp'> & {
  issuer?: string
}

type MFAEnrollPhoneParams = MFAEnrollParamsBase<'phone'> & {
  phone: string
}

type MFAEnrollWebauthnParams = MFAEnrollParamsBase<'webauthn'>
```

**Purpose:** Start enrollment for a new MFA factor (TOTP, phone, or WebAuthn). Creates an unverified factor.

**Parameters:**

- `params.factorType` (`'totp' | 'phone' | 'webauthn'`) - Type of factor to enroll
- `params.friendlyName?` (string) - Human-readable name for the factor
- `params.issuer?` (string) - TOTP only: domain the user is enrolled with
- `params.phone?` (string) - Phone only: E.164 format phone number

**Returns:**

```ts
// TOTP
{
  data: {
    id: string              // Factor ID (unverified)
    type: 'totp'
    friendly_name?: string
    totp: {
      qr_code: string       // SVG QR code (data URI after client transform)
      secret: string         // TOTP secret for manual entry
      uri: string            // otpauth:// URI
    }
  },
  error: AuthError | null
}

// Phone
{
  data: {
    id: string
    type: 'phone'
    friendly_name?: string
    phone: string            // E.164 phone number
  },
  error: AuthError | null
}

// WebAuthn
{
  data: {
    id: string
    type: 'webauthn'
    friendly_name?: string
  },
  error: AuthError | null
}
```

**Usage:**

```ts
// Enroll TOTP factor
const { data, error } = await client.auth.mfa.enroll({
  factorType: 'totp',
  friendlyName: 'My Authenticator',
  issuer: 'myapp.com',
})

if (data) {
  // Display QR code to user
  const img = document.createElement('img')
  img.src = data.totp.qr_code
  document.body.appendChild(img)

  // Or show secret for manual entry
  console.log('Secret:', data.totp.secret)
}

// Enroll phone factor
const { data, error } = await client.auth.mfa.enroll({
  factorType: 'phone',
  friendlyName: 'My Phone',
  phone: '+12025551234',
})

// Enroll WebAuthn factor (server-side only, no browser interaction)
const { data, error } = await client.auth.mfa.enroll({
  factorType: 'webauthn',
  friendlyName: 'My Security Key',
})
```

**Implementation Considerations:**

- **Backend Requirements:** Factor storage, TOTP secret generation, QR code SVG generation
- **Security:** Factor created in `unverified` state; must call `verify()` to activate. Upon verification all other sessions logged out, session promoted to `aal2`
- **Complexity:** Moderate - secret generation for TOTP, phone validation for phone, credential setup for WebAuthn
- **Dependencies:** TOTP library (secret + QR), SMS provider (phone), WebAuthn server library
- **Storage:** No client-side storage changes until factor is verified

**Authentication Flow:**

```
mfa.enroll({ factorType: 'totp', friendlyName: 'My App' })
  → POST /factors
      → Headers: Authorization: Bearer <jwt>
      → Body: { factor_type: 'totp', friendly_name: 'My App', issuer: 'myapp.com' }
  → Backend:
      → Validate JWT, get user
      → Generate TOTP secret
      → Generate QR code SVG
      → Create factor record (status: 'unverified')
      → Return factor ID + TOTP details
  → Client:
      → Prepends data:image/svg+xml;utf-8, to qr_code
      → Display QR/secret to user
  → User adds to authenticator app
  → Call mfa.verify() with code from app
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /factors`
- **Headers:** `Authorization: Bearer <jwt>`
- **Request Body:**

```json
{
  "factor_type": "totp",
  "friendly_name": "My Authenticator",
  "issuer": "myapp.com"
}
```

- **Response (TOTP):**

```json
{
  "id": "factor-uuid",
  "type": "totp",
  "friendly_name": "My Authenticator",
  "totp": {
    "qr_code": "<svg>...</svg>",
    "secret": "JBSWY3DPEHPK3PXP",
    "uri": "otpauth://totp/myapp.com:user@example.com?secret=JBSWY3DPEHPK3PXP&issuer=myapp.com"
  }
}
```

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `unauthorized` - invalid JWT
- `factor_already_exists` - duplicate factor enrollment
- `too_many_enrolled_factors` - exceeded max enrolled factors

**Related:** `mfa.challenge`, `mfa.verify`, `mfa.listFactors`

---

### `mfa.webauthn.enroll(params)`

**Priority:** Medium | **Complexity:** Simple (wrapper)

**Signature:**

```ts
async mfa.webauthn.enroll(
  params: Omit<MFAEnrollWebauthnParams, 'factorType'>
): Promise<AuthMFAEnrollWebauthnResponse>
```

**Purpose:** Convenience wrapper that calls `mfa.enroll()` with `factorType: 'webauthn'` preset.

**Parameters:**

- `params.friendlyName?` (string) - Human-readable name for the WebAuthn factor

**Returns:**

```ts
{
  data: {
    id: string
    type: 'webauthn'
    friendly_name?: string
  },
  error: AuthError | null
}
```

**Usage:**

```ts
const { data, error } = await client.auth.mfa.webauthn.enroll({
  friendlyName: 'My Security Key',
})

if (data) {
  console.log('Factor enrolled:', data.id)
  // Must still challenge + verify to activate
}
```

**Implementation Considerations:**

- **Backend Requirements:** Same as `mfa.enroll` with `factorType: 'webauthn'`
- **Security:** Creates unverified factor; no browser credential interaction at this step
- **Complexity:** Simple - delegates entirely to `mfa.enroll`
- **Dependencies:** None beyond `mfa.enroll`
- **Storage:** No client-side changes

**Authentication Flow:**

```
mfa.webauthn.enroll({ friendlyName: 'My Key' })
  → Calls mfa.enroll({ factorType: 'webauthn', friendlyName: 'My Key' })
  → POST /factors
      → Body: { factor_type: 'webauthn', friendly_name: 'My Key' }
  → Returns unverified factor
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /factors`
- **Request Body:**

```json
{
  "factor_type": "webauthn",
  "friendly_name": "My Security Key"
}
```

- **Response:**

```json
{
  "id": "factor-uuid",
  "type": "webauthn",
  "friendly_name": "My Security Key"
}
```

**Error Cases:**

- Same as `mfa.enroll`

**Related:** `mfa.enroll`, `mfa.webauthn.register`, `mfa.webauthn.challenge`

---

## Challenges

### `mfa.challenge(params)`

**Priority:** Medium | **Complexity:** Moderate

**Signature:**

```ts
// TOTP
async mfa.challenge(params: MFAChallengeTOTPParams): Promise<AuthMFAChallengeTOTPResponse>

// Phone
async mfa.challenge(params: MFAChallengePhoneParams): Promise<AuthMFAChallengePhoneResponse>

// WebAuthn
async mfa.challenge(params: MFAChallengeWebauthnParams): Promise<AuthMFAChallengeWebauthnResponse>

// Union
async mfa.challenge(params: MFAChallengeParams): Promise<AuthMFAChallengeResponse>

// Param types
type MFAChallengeParamsBase = {
  factorId: string
}

type MFAChallengeTOTPParams = MFAChallengeParamsBase

type MFAChallengePhoneParams = MFAChallengeParamsBase & {
  channel: 'sms' | 'whatsapp'
}

type MFAChallengeWebauthnParams = MFAChallengeParamsBase & {
  webauthn: {
    rpId: string
    rpOrigins?: string[]
  }
}
```

**Purpose:** Create a challenge to verify user has access to an MFA factor. Required step before `mfa.verify()`.

**Parameters:**

- `params.factorId` (string) - ID of the factor to challenge (from `enroll()`)
- `params.channel?` (`'sms' | 'whatsapp'`) - Phone only: messaging channel
- `params.webauthn?` (object) - WebAuthn only:
  - `params.webauthn.rpId` (string) - Relying Party ID (hostname)
  - `params.webauthn.rpOrigins?` (string[]) - Allowed origins

**Returns:**

```ts
// TOTP
{
  data: {
    id: string           // Challenge ID
    type: 'totp'
    expires_at: number   // UNIX timestamp
  },
  error: AuthError | null
}

// Phone
{
  data: {
    id: string
    type: 'phone'
    expires_at: number
  },
  error: AuthError | null
}

// WebAuthn
{
  data: {
    id: string
    type: 'webauthn'
    expires_at: number
    webauthn:
      | {
          type: 'create'
          credential_options: { publicKey: PublicKeyCredentialCreationOptions }
        }
      | {
          type: 'request'
          credential_options: { publicKey: PublicKeyCredentialRequestOptions }
        }
  },
  error: AuthError | null
}
```

**Usage:**

```ts
// TOTP challenge
const { data, error } = await client.auth.mfa.challenge({
  factorId: 'factor-uuid',
})

if (data) {
  // Prompt user for TOTP code, then call mfa.verify()
  const code = prompt('Enter your 2FA code:')
  await client.auth.mfa.verify({
    factorId: 'factor-uuid',
    challengeId: data.id,
    code,
  })
}

// Phone challenge (send SMS)
const { data, error } = await client.auth.mfa.challenge({
  factorId: 'phone-factor-uuid',
  channel: 'sms',
})

// WebAuthn challenge
const { data, error } = await client.auth.mfa.challenge({
  factorId: 'webauthn-factor-uuid',
  webauthn: {
    rpId: window.location.hostname,
    rpOrigins: [window.location.origin],
  },
})
// Returns credential_options for browser WebAuthn API
```

**Implementation Considerations:**

- **Backend Requirements:** Challenge generation, factor validation, WebAuthn credential options generation
- **Security:** Challenges expire (see `expires_at`). Acquires lock to prevent concurrent challenges.
- **Complexity:** Moderate - TOTP/phone are simple; WebAuthn requires deserializing server JSON to browser-native credential options (base64url to ArrayBuffer)
- **Dependencies:** WebAuthn: browser `PublicKeyCredential` API support
- **Storage:** No client storage changes

**Authentication Flow:**

```
mfa.challenge({ factorId: 'xxx' })
  → Acquire lock
  → POST /factors/{factorId}/challenge
      → Headers: Authorization: Bearer <jwt>
      → Body: { factorId: 'xxx' } (+ webauthn params if applicable)
  → Backend:
      → Validate JWT, verify factor belongs to user
      → Generate challenge (nonce/timestamp)
      → For WebAuthn: generate credential creation/request options
      → Return challenge ID + options
  → Client (WebAuthn):
      → Deserialize JSON credential options (base64url to ArrayBuffer)
      → Uses deserializeCredentialCreationOptions() or deserializeCredentialRequestOptions()
      → Supports native WebAuthn Level 3 parseCreationOptionsFromJSON with manual fallback
  → Return challenge data
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /factors/{factorId}/challenge`
- **Headers:** `Authorization: Bearer <jwt>`
- **Request Body (TOTP):**

```json
{
  "factorId": "factor-uuid"
}
```

- **Request Body (WebAuthn):**

```json
{
  "factorId": "factor-uuid",
  "webauthn": {
    "rpId": "myapp.com",
    "rpOrigins": ["https://myapp.com"]
  }
}
```

- **Response (TOTP):**

```json
{
  "id": "challenge-uuid",
  "type": "totp",
  "expires_at": 1704067200
}
```

- **Response (WebAuthn create):**

```json
{
  "id": "challenge-uuid",
  "type": "webauthn",
  "expires_at": 1704067200,
  "webauthn": {
    "type": "create",
    "credential_options": {
      "publicKey": {
        "challenge": "<base64url>",
        "rp": { "name": "myapp", "id": "myapp.com" },
        "user": { "id": "<base64url>", "name": "user@example.com", "displayName": "User" },
        "pubKeyCredParams": [{ "type": "public-key", "alg": -7 }],
        "excludeCredentials": [],
        "authenticatorSelection": {},
        "attestation": "direct"
      }
    }
  }
}
```

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `factor_not_found` - invalid factor ID
- `unauthorized` - factor doesn't belong to user

**Related:** `mfa.enroll`, `mfa.verify`, `mfa.challengeAndVerify`

---

### `mfa.webauthn.challenge(params, overrides?)`

**Priority:** Medium | **Complexity:** Complex

**Signature:**

```ts
async mfa.webauthn.challenge(
  params: MFAChallengeWebauthnParams & {
    friendlyName?: string
    signal?: AbortSignal
  },
  overrides?: {
    create?: Partial<PublicKeyCredentialCreationOptions>
    request?: never
  } | {
    create?: never
    request?: Partial<PublicKeyCredentialRequestOptions>
  }
): Promise<RequestResult<
  { factorId: string; challengeId: string } & {
    webauthn: {
      type: 'create' | 'request'
      credential_response: RegistrationCredential | AuthenticationCredential
    }
  },
  WebAuthnError | AuthError
>>
```

**Purpose:** Combined server challenge + browser credential operation. Gets challenge from server, then invokes browser `navigator.credentials.create()` or `.get()` depending on factor status.

**Parameters:**

- `params.factorId` (string) - Factor ID to challenge
- `params.webauthn` (object) - `{ rpId: string, rpOrigins?: string[] }`
- `params.friendlyName?` (string) - Sets user name/displayName on credential if blank
- `params.signal?` (AbortSignal) - Abort signal; defaults to `WebAuthnAbortService` singleton
- `overrides?.create?` (Partial\<PublicKeyCredentialCreationOptions\>) - Override credential creation options
- `overrides?.request?` (Partial\<PublicKeyCredentialRequestOptions\>) - Override credential request options

**Returns:**

```ts
{
  data: {
    factorId: string
    challengeId: string
    webauthn: {
      type: 'create' | 'request'
      credential_response: RegistrationCredential | AuthenticationCredential
    }
  },
  error: WebAuthnError | AuthError | null
}
```

**Usage:**

```ts
// Challenge for unverified factor (create flow)
const { data, error } = await client.auth.mfa.webauthn.challenge({
  factorId: 'factor-uuid',
  webauthn: {
    rpId: window.location.hostname,
    rpOrigins: [window.location.origin],
  },
  friendlyName: 'My Key',
})
// Browser prompts user to insert/touch security key
// Returns credential_response for mfa.webauthn.verify()

// With custom overrides
const { data, error } = await client.auth.mfa.webauthn.challenge(
  {
    factorId: 'factor-uuid',
    webauthn: { rpId: 'myapp.com' },
  },
  {
    create: {
      authenticatorSelection: {
        authenticatorAttachment: 'platform',
        userVerification: 'required',
      },
    },
  }
)

// With abort signal
const controller = new AbortController()
const { data, error } = await client.auth.mfa.webauthn.challenge({
  factorId: 'factor-uuid',
  webauthn: { rpId: 'myapp.com' },
  signal: controller.signal,
})
// Call controller.abort() to cancel
```

**Implementation Considerations:**

- **Backend Requirements:** Same as `mfa.challenge` for server side
- **Security:** Only one WebAuthn ceremony active at a time (WebAuthnAbortService cancels previous). Default options: `userVerification: 'preferred'`, `authenticatorAttachment: 'cross-platform'`, `attestation: 'direct'`
- **Complexity:** Complex - orchestrates server challenge, option deserialization, browser credential API, and serialization
- **Dependencies:** Browser WebAuthn API (`navigator.credentials`), `PublicKeyCredential`
- **Storage:** No client storage; credential stays in memory until `verify()`

**Authentication Flow:**

```
mfa.webauthn.challenge({ factorId, webauthn: { rpId, rpOrigins } })
  → Call mfa.challenge({ factorId, webauthn }) → server returns challenge + credential_options
  → If type === 'create':
      → Merge DEFAULT_CREATION_OPTIONS + server options + overrides
      → navigator.credentials.create({ publicKey, signal })
      → Return { factorId, challengeId, webauthn: { type: 'create', credential_response } }
  → If type === 'request':
      → Merge DEFAULT_REQUEST_OPTIONS + server options + overrides
      → navigator.credentials.get({ publicKey, signal })
      → Return { factorId, challengeId, webauthn: { type: 'request', credential_response } }
```

**Default Credential Options:**

```ts
// Creation defaults (for unverified factors)
{
  hints: ['security-key'],
  authenticatorSelection: {
    authenticatorAttachment: 'cross-platform',
    requireResidentKey: false,
    userVerification: 'preferred',
    residentKey: 'discouraged',
  },
  attestation: 'direct',
}

// Request defaults (for verified factors)
{
  userVerification: 'preferred',
  hints: ['security-key'],
  attestation: 'direct',
}
```

**GoTrue Backend Mapping:**

- Same endpoint as `mfa.challenge`: `POST /factors/{factorId}/challenge`
- Client-side browser interaction adds the credential response layer

**Error Cases:**

- All `mfa.challenge` errors
- `WebAuthnError` - browser credential operation failed (user cancelled, timeout, not supported)
- `WebAuthnUnknownError` - empty or unexpected credential response from browser
- `AbortError` - ceremony cancelled by abort signal or new concurrent ceremony

**Related:** `mfa.challenge`, `mfa.webauthn.verify`, `mfa.webauthn.register`, `mfa.webauthn.authenticate`

---

## Verification

### `mfa.verify(params)`

**Priority:** Medium | **Complexity:** Moderate

**Signature:**

```ts
// TOTP
async mfa.verify(params: MFAVerifyTOTPParams): Promise<AuthMFAVerifyResponse>

// Phone
async mfa.verify(params: MFAVerifyPhoneParams): Promise<AuthMFAVerifyResponse>

// WebAuthn
async mfa.verify<T extends 'create' | 'request'>(
  params: MFAVerifyWebauthnParams<T>
): Promise<AuthMFAVerifyResponse>

// Union
async mfa.verify(params: MFAVerifyParams): Promise<AuthMFAVerifyResponse>

// Param types
type MFAVerifyParamsBase = {
  factorId: string
  challengeId: string
}

type MFAVerifyTOTPParams = MFAVerifyParamsBase & {
  code: string
}

type MFAVerifyWebauthnParams<T extends 'create' | 'request'> = MFAVerifyParamsBase & {
  webauthn: {
    rpId: string
    rpOrigins?: string[]
    type: T
    credential_response: T extends 'create' ? RegistrationCredential : AuthenticationCredential
  }
}
```

**Purpose:** Verify an MFA challenge. Completes factor verification or step-up authentication, promoting session to `aal2`.

**Parameters:**

- `params.factorId` (string) - Factor ID (from `enroll()`)
- `params.challengeId` (string) - Challenge ID (from `challenge()`)
- `params.code?` (string) - TOTP/Phone: verification code from authenticator app or SMS
- `params.webauthn?` (object) - WebAuthn:
  - `webauthn.rpId` (string) - Relying Party ID
  - `webauthn.rpOrigins?` (string[]) - Allowed origins
  - `webauthn.type` (`'create' | 'request'`) - Operation type
  - `webauthn.credential_response` (RegistrationCredential | AuthenticationCredential) - Browser credential

**Returns:**

```ts
{
  data: {
    access_token: string     // New JWT with aal2
    token_type: 'bearer'
    expires_in: number       // Seconds until expiry
    refresh_token: string
    user: User
  },
  error: AuthError | null
}
```

**Usage:**

```ts
// Verify TOTP
const { data, error } = await client.auth.mfa.verify({
  factorId: 'factor-uuid',
  challengeId: 'challenge-uuid',
  code: '123456',
})

if (data) {
  // Session upgraded to aal2
  console.log('MFA verified, new token:', data.access_token)
}

// Verify WebAuthn (after webauthn.challenge)
const { data: challengeData } = await client.auth.mfa.webauthn.challenge({
  factorId: 'factor-uuid',
  webauthn: { rpId: window.location.hostname },
})

const { data, error } = await client.auth.mfa.verify({
  factorId: 'factor-uuid',
  challengeId: challengeData.challengeId,
  webauthn: {
    type: challengeData.webauthn.type,
    rpId: window.location.hostname,
    rpOrigins: [window.location.origin],
    credential_response: challengeData.webauthn.credential_response,
  },
})
```

**Implementation Considerations:**

- **Backend Requirements:** Challenge validation, TOTP code verification, WebAuthn credential verification, session token generation
- **Security:** Acquires lock. On success: saves new session, notifies `MFA_CHALLENGE_VERIFIED` event, session promoted to `aal2`. WebAuthn credentials serialized before sending (binary to base64url JSON).
- **Complexity:** Moderate - TOTP is simple code check; WebAuthn requires serializing `RegistrationCredential`/`AuthenticationCredential` to JSON (uses `serializeCredentialCreationResponse` or `serializeCredentialRequestResponse`)
- **Dependencies:** Session storage, event system
- **Storage:** Saves new session with updated tokens and `aal2` claim

**Authentication Flow:**

```
mfa.verify({ factorId, challengeId, code })
  → Acquire lock
  → POST /factors/{factorId}/verify
      → Headers: Authorization: Bearer <jwt>
      → Body (TOTP): { challenge_id: 'xxx', code: '123456' }
      → Body (WebAuthn): {
            challenge_id: 'xxx',
            webauthn: {
              type: 'create',
              rpId: 'myapp.com',
              credential_response: { id, rawId, response: {...}, type: 'public-key' }
            }
          }
  → Backend:
      → Validate challenge not expired
      → TOTP: verify code against secret
      → WebAuthn: verify credential against stored public key
      → Generate new session tokens with aal2
      → Return tokens + user
  → Client:
      → Save session (expires_at calculated from expires_in)
      → Notify MFA_CHALLENGE_VERIFIED event
```

**GoTrue Backend Mapping:**

- **Endpoint:** `POST /factors/{factorId}/verify`
- **Headers:** `Authorization: Bearer <jwt>`
- **Request Body (TOTP):**

```json
{
  "challenge_id": "challenge-uuid",
  "code": "123456"
}
```

- **Request Body (WebAuthn):**

```json
{
  "challenge_id": "challenge-uuid",
  "webauthn": {
    "type": "create",
    "rpId": "myapp.com",
    "rpOrigins": ["https://myapp.com"],
    "credential_response": {
      "id": "credential-id",
      "rawId": "credential-id",
      "response": {
        "attestationObject": "<base64url>",
        "clientDataJSON": "<base64url>"
      },
      "type": "public-key",
      "clientExtensionResults": {},
      "authenticatorAttachment": "cross-platform"
    }
  }
}
```

- **Response:**

```json
{
  "access_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 3600,
  "refresh_token": "xxx",
  "user": { "id": "uuid", "factors": [], "..." : "..." }
}
```

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `challenge_expired` - challenge past `expires_at`
- `invalid_code` - wrong TOTP/phone code
- `factor_not_found` - invalid factor ID
- `unauthorized` - invalid JWT

**Related:** `mfa.enroll`, `mfa.challenge`, `mfa.challengeAndVerify`

---

### `mfa.webauthn.verify(params)`

**Priority:** Medium | **Complexity:** Simple (wrapper)

**Signature:**

```ts
async mfa.webauthn.verify<T extends 'create' | 'request'>(params: {
  challengeId: string
  factorId: string
  webauthn: MFAVerifyWebauthnParams<T>['webauthn']
}): Promise<AuthMFAVerifyResponse>
```

**Purpose:** Convenience wrapper that calls `mfa.verify()` for WebAuthn credentials. Used after `mfa.webauthn.challenge()`.

**Parameters:**

- `params.factorId` (string) - Factor ID
- `params.challengeId` (string) - Challenge ID from `webauthn.challenge()`
- `params.webauthn` (object) - WebAuthn verification fields (same as `mfa.verify` webauthn param)

**Returns:**

```ts
// Same as mfa.verify
{
  data: {
    access_token: string
    token_type: 'bearer'
    expires_in: number
    refresh_token: string
    user: User
  },
  error: AuthError | null
}
```

**Usage:**

```ts
// After getting challenge data from webauthn.challenge()
const { data, error } = await client.auth.mfa.webauthn.verify({
  factorId: 'factor-uuid',
  challengeId: challengeData.challengeId,
  webauthn: {
    type: challengeData.webauthn.type,
    rpId: window.location.hostname,
    rpOrigins: [window.location.origin],
    credential_response: challengeData.webauthn.credential_response,
  },
})
```

**Implementation Considerations:**

- **Backend Requirements:** Same as `mfa.verify` with WebAuthn params
- **Security:** Delegates to `mfa.verify` which handles lock acquisition and session save
- **Complexity:** Simple - thin wrapper
- **Dependencies:** `mfa.verify`
- **Storage:** Same as `mfa.verify` (session saved on success)

**Authentication Flow:**

```
mfa.webauthn.verify({ factorId, challengeId, webauthn })
  → Calls mfa.verify({ factorId, challengeId, webauthn })
  → Same flow as mfa.verify (WebAuthn path)
```

**GoTrue Backend Mapping:**

- Same as `mfa.verify` WebAuthn path: `POST /factors/{factorId}/verify`

**Error Cases:**

- Same as `mfa.verify`

**Related:** `mfa.webauthn.challenge`, `mfa.verify`, `mfa.webauthn.authenticate`

---

### `mfa.challengeAndVerify(params)`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async mfa.challengeAndVerify(
  params: MFAChallengeAndVerifyParams
): Promise<AuthMFAVerifyResponse>

type MFAChallengeAndVerifyParams = {
  factorId: string
  code: string
}
```

**Purpose:** Combined challenge + verify in a single call. TOTP only (no WebAuthn variant). Convenience method that creates a challenge then immediately verifies with provided code.

**Parameters:**

- `params.factorId` (string) - Factor ID to challenge and verify
- `params.code` (string) - TOTP verification code

**Returns:**

```ts
{
  data: {
    access_token: string
    token_type: 'bearer'
    expires_in: number
    refresh_token: string
    user: User
  },
  error: AuthError | null
}
```

**Usage:**

```ts
// One-step MFA verification
const code = prompt('Enter your 2FA code:')

const { data, error } = await client.auth.mfa.challengeAndVerify({
  factorId: 'factor-uuid',
  code,
})

if (data) {
  console.log('MFA complete, session upgraded to aal2')
}

// Complete step-up auth flow
const { data: aalData } = await client.auth.mfa.getAuthenticatorAssuranceLevel()

if (aalData.currentLevel === 'aal1' && aalData.nextLevel === 'aal2') {
  const { data: factors } = await client.auth.mfa.listFactors()
  const totpFactor = factors.totp[0]

  const code = prompt('Enter your 2FA code:')
  const { data, error } = await client.auth.mfa.challengeAndVerify({
    factorId: totpFactor.id,
    code,
  })
}
```

**Implementation Considerations:**

- **Backend Requirements:** Same as `challenge` + `verify` combined
- **Security:** Both `_challenge` and `_verify` independently acquire locks; no additional lock needed
- **Complexity:** Simple - sequential call to `_challenge` then `_verify`
- **Dependencies:** `mfa.challenge`, `mfa.verify`
- **Storage:** Session saved after verify step

**Authentication Flow:**

```
mfa.challengeAndVerify({ factorId, code })
  → _challenge({ factorId })
      → POST /factors/{factorId}/challenge → returns { id: challengeId }
  → _verify({ factorId, challengeId, code })
      → POST /factors/{factorId}/verify → returns session tokens
  → Session saved, MFA_CHALLENGE_VERIFIED event
```

**GoTrue Backend Mapping:**

- Two sequential requests:
  1. `POST /factors/{factorId}/challenge` (same as `mfa.challenge`)
  2. `POST /factors/{factorId}/verify` (same as `mfa.verify`)

**Error Cases:**

- All `mfa.challenge` errors (first step)
- All `mfa.verify` errors (second step)

**Related:** `mfa.challenge`, `mfa.verify`, `mfa.listFactors`

---

## Management

### `mfa.unenroll(params)`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async mfa.unenroll(params: MFAUnenrollParams): Promise<AuthMFAUnenrollResponse>

type MFAUnenrollParams = {
  factorId: string
}
```

**Purpose:** Remove an enrolled MFA factor. Requires `aal2` session to unenroll a verified factor.

**Parameters:**

- `params.factorId` (string) - ID of the factor to remove

**Returns:**

```ts
{
  data: {
    id: string     // ID of the removed factor
  },
  error: AuthError | null
}
```

**Usage:**

```ts
// Remove a factor
const { data, error } = await client.auth.mfa.unenroll({
  factorId: 'factor-uuid',
})

if (data) {
  console.log('Factor removed:', data.id)
}

// Remove all TOTP factors
const { data: factors } = await client.auth.mfa.listFactors()
for (const factor of factors.totp) {
  await client.auth.mfa.unenroll({ factorId: factor.id })
}
```

**Implementation Considerations:**

- **Backend Requirements:** Factor deletion, session validation
- **Security:** Must have `aal2` session to unenroll verified factors. Unverified factors can be removed at `aal1`.
- **Complexity:** Simple - single DELETE request
- **Dependencies:** Active session
- **Storage:** No client storage changes (factor removed server-side)

**Authentication Flow:**

```
mfa.unenroll({ factorId: 'xxx' })
  → DELETE /factors/{factorId}
      → Headers: Authorization: Bearer <jwt>
  → Backend:
      → Validate JWT
      → Verify aal2 if factor is verified
      → Delete factor record
      → Return deleted factor ID
```

**GoTrue Backend Mapping:**

- **Endpoint:** `DELETE /factors/{factorId}`
- **Headers:** `Authorization: Bearer <jwt>`
- **Response:**

```json
{
  "id": "factor-uuid"
}
```

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `factor_not_found` - invalid factor ID
- `insufficient_aal` - trying to unenroll verified factor without `aal2`
- `unauthorized` - invalid JWT

**Related:** `mfa.enroll`, `mfa.listFactors`

---

### `mfa.listFactors()`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async mfa.listFactors(): Promise<AuthMFAListFactorsResponse>

type AuthMFAListFactorsResponse = RequestResult<{
  all: Factor[]
  totp: Factor<'totp', 'verified'>[]
  phone: Factor<'phone', 'verified'>[]
  webauthn: Factor<'webauthn', 'verified'>[]
}>

type Factor<Type = FactorType, Status = FactorVerificationStatus> = {
  id: string
  friendly_name?: string
  factor_type: Type
  status: Status
  created_at: string
  updated_at: string
  last_challenged_at?: string
}
```

**Purpose:** List all MFA factors for the current user. Returns both verified and unverified factors in `all`, only verified factors in typed arrays.

**Parameters:** None

**Returns:**

```ts
{
  data: {
    all: Factor[]                          // All factors (verified + unverified)
    totp: Factor<'totp', 'verified'>[]     // Verified TOTP factors only
    phone: Factor<'phone', 'verified'>[]   // Verified phone factors only
    webauthn: Factor<'webauthn', 'verified'>[] // Verified WebAuthn factors only
  },
  error: AuthError | null
}
```

**Usage:**

```ts
const { data, error } = await client.auth.mfa.listFactors()

if (data) {
  console.log('All factors:', data.all.length)
  console.log('Verified TOTP:', data.totp.length)
  console.log('Verified phone:', data.phone.length)
  console.log('Verified WebAuthn:', data.webauthn.length)
}

// Check if user has any MFA set up
const hasMFA = data.totp.length > 0 || data.phone.length > 0 || data.webauthn.length > 0

// Get first verified TOTP factor for challengeAndVerify
if (data.totp.length > 0) {
  const { data: verifyData } = await client.auth.mfa.challengeAndVerify({
    factorId: data.totp[0].id,
    code: '123456',
  })
}
```

**Implementation Considerations:**

- **Backend Requirements:** Reads from `user.factors` (no dedicated endpoint)
- **Security:** Calls `getUser()` which acquires lock and validates session
- **Complexity:** Simple - reads user object and categorizes factors in single loop
- **Dependencies:** `getUser()`
- **Storage:** No storage changes; reads from user object

**Authentication Flow:**

```
mfa.listFactors()
  → getUser() (acquires lock, GET /user)
  → Loop user.factors once:
      → Push all to data.all
      → Push verified to data[factor.factor_type]
  → Return categorized factors
```

**GoTrue Backend Mapping:**

- **Endpoint:** Uses `GET /user` (via `getUser()`)
- Factors are a field on the User object, not a separate endpoint

**Error Cases:**

- `AuthSessionMissingError` - no active session
- `unauthorized` - invalid JWT

**Related:** `mfa.enroll`, `mfa.unenroll`, `mfa.getAuthenticatorAssuranceLevel`

---

### `mfa.getAuthenticatorAssuranceLevel()`

**Priority:** Medium | **Complexity:** Simple

**Signature:**

```ts
async mfa.getAuthenticatorAssuranceLevel(): Promise<AuthMFAGetAuthenticatorAssuranceLevelResponse>

type AuthMFAGetAuthenticatorAssuranceLevelResponse = RequestResult<{
  currentLevel: AuthenticatorAssuranceLevels | null
  nextLevel: AuthenticatorAssuranceLevels | null
  currentAuthenticationMethods: AMREntry[] | string[]
}>

type AuthenticatorAssuranceLevels = 'aal1' | 'aal2'
```

**Purpose:** Get current Authenticator Assurance Level (AAL) for the session. Use to determine if user needs MFA step-up.

**Parameters:** None

**Returns:**

```ts
{
  data: {
    currentLevel: 'aal1' | 'aal2' | null    // Current session AAL
    nextLevel: 'aal1' | 'aal2' | null       // Highest achievable AAL
    currentAuthenticationMethods: AMREntry[] | string[]  // Methods used in session
  },
  error: AuthError | null
}
```

**Usage:**

```ts
const { data, error } = await client.auth.mfa.getAuthenticatorAssuranceLevel()

if (data) {
  // Check if MFA step-up needed
  if (data.currentLevel === 'aal1' && data.nextLevel === 'aal2') {
    // User has verified MFA factors but hasn't used them this session
    // Show MFA verification screen
    showMFAScreen()
  }

  // Already at aal2
  if (data.currentLevel === 'aal2') {
    // Full access granted
  }

  // No MFA factors enrolled
  if (data.nextLevel === 'aal1') {
    // User hasn't set up MFA
    showMFAEnrollmentPrompt()
  }
}

// Check authentication methods
const { data } = await client.auth.mfa.getAuthenticatorAssuranceLevel()
console.log('Auth methods:', data.currentAuthenticationMethods)
// e.g. [{ method: 'password', timestamp: 1704067200 }, { method: 'totp', timestamp: 1704067260 }]
// or ['password', 'otp'] (RFC-8176 format)
```

**Implementation Considerations:**

- **Backend Requirements:** None - decoded from JWT client-side
- **Security:** Reads `aal` and `amr` claims from JWT. No network call in most cases.
- **Complexity:** Simple - JWT decode + factor check
- **Dependencies:** `getSession()`, JWT decoder
- **Storage:** No storage changes; reads existing session

**Authentication Flow:**

```
mfa.getAuthenticatorAssuranceLevel()
  → getSession() → get current session
  → If no session: return { currentLevel: null, nextLevel: null, methods: [] }
  → Decode JWT access_token:
      → Read payload.aal → currentLevel
      → Read payload.amr → currentAuthenticationMethods
  → Check session.user.factors:
      → If any verified factors exist → nextLevel = 'aal2'
      → Else nextLevel = currentLevel
  → Return { currentLevel, nextLevel, currentAuthenticationMethods }
```

**GoTrue Backend Mapping:**

- No backend call - purely client-side JWT inspection
- Uses `getSession()` which may refresh token if expired

**Error Cases:**

- `AuthSessionMissingError` - `getSession()` fails
- Returns `{ currentLevel: null, nextLevel: null, currentAuthenticationMethods: [] }` when no session (not an error)

**Related:** `mfa.listFactors`, `mfa.challenge`, `mfa.challengeAndVerify`

---

## WebAuthn

### `mfa.webauthn.authenticate(params, overrides?)`

**Priority:** Medium | **Complexity:** Complex

**Signature:**

```ts
async mfa.webauthn.authenticate(
  params: {
    factorId: string
    webauthn?: {
      rpId?: string        // Defaults to window.location.hostname
      rpOrigins?: string[] // Defaults to [window.location.origin]
      signal?: AbortSignal
    }
  },
  overrides?: PublicKeyCredentialRequestOptions
): Promise<RequestResult<AuthMFAVerifyResponseData, WebAuthnError | AuthError>>
```

**Purpose:** Complete WebAuthn authentication flow in a single call. Performs challenge + browser credential request + verify for an existing verified factor. Used for step-up authentication.

**Parameters:**

- `params.factorId` (string) - ID of the verified WebAuthn factor
- `params.webauthn?.rpId?` (string) - Relying Party ID; defaults to `window.location.hostname`
- `params.webauthn?.rpOrigins?` (string[]) - Allowed origins; defaults to `[window.location.origin]`
- `params.webauthn?.signal?` (AbortSignal) - Optional abort signal
- `overrides?` (PublicKeyCredentialRequestOptions) - Override `navigator.credentials.get()` options

**Returns:**

```ts
{
  data: {
    access_token: string
    token_type: 'bearer'
    expires_in: number
    refresh_token: string
    user: User
  },
  error: WebAuthnError | AuthError | null
}
```

**Usage:**

```ts
// Basic authentication with defaults
const { data, error } = await client.auth.mfa.webauthn.authenticate({
  factorId: 'webauthn-factor-uuid',
})
// Browser prompts for security key touch
// On success, session upgraded to aal2

// With explicit RP config
const { data, error } = await client.auth.mfa.webauthn.authenticate({
  factorId: 'webauthn-factor-uuid',
  webauthn: {
    rpId: 'myapp.com',
    rpOrigins: ['https://myapp.com', 'https://www.myapp.com'],
  },
})

// With overrides for credential request
const { data, error } = await client.auth.mfa.webauthn.authenticate(
  { factorId: 'webauthn-factor-uuid' },
  { userVerification: 'required' }
)

// Full step-up auth flow
const { data: factors } = await client.auth.mfa.listFactors()
const webauthnFactor = factors.webauthn[0]

if (webauthnFactor) {
  const { data, error } = await client.auth.mfa.webauthn.authenticate({
    factorId: webauthnFactor.id,
  })
  if (data) {
    console.log('MFA complete via WebAuthn')
  }
}
```

**Implementation Considerations:**

- **Backend Requirements:** WebAuthn challenge generation, credential verification
- **Security:** Checks `browserSupportsWebAuthn()` first. Returns error if `rpId` is falsy. Uses `webauthn.challenge()` with `request` overrides, then `webauthn.verify()`.
- **Complexity:** Complex - orchestrates full ceremony: browser support check, challenge (server + browser credential.get()), verify
- **Dependencies:** Browser WebAuthn API, `navigator.credentials.get()`
- **Storage:** Session saved after successful verification

**Authentication Flow:**

```
mfa.webauthn.authenticate({ factorId, webauthn: { rpId, rpOrigins, signal } })
  → Validate rpId exists (error if not)
  → Check browserSupportsWebAuthn() (error if not)
  → webauthn.challenge({ factorId, webauthn: { rpId, rpOrigins }, signal }, { request: overrides })
      → POST /factors/{factorId}/challenge → server returns credential request options
      → navigator.credentials.get({ publicKey, signal }) → user touches key
      → Returns { challengeId, webauthn: { type: 'request', credential_response } }
  → webauthn.verify({ factorId, challengeId, webauthn: { type, rpId, rpOrigins, credential_response } })
      → POST /factors/{factorId}/verify → server verifies credential
      → Returns new session tokens (aal2)
  → Session saved, MFA_CHALLENGE_VERIFIED event
```

**GoTrue Backend Mapping:**

- Two sequential requests:
  1. `POST /factors/{factorId}/challenge` with WebAuthn params
  2. `POST /factors/{factorId}/verify` with serialized credential response

**Error Cases:**

- `AuthError('rpId is required for WebAuthn authentication')` - rpId is falsy
- `AuthUnknownError('Browser does not support WebAuthn')` - missing WebAuthn APIs
- All `mfa.webauthn.challenge` errors
- All `mfa.webauthn.verify` errors
- `WebAuthnError` - user cancelled, timeout, credential not recognized
- `AuthUnknownError('Unexpected error in authenticate')` - catch-all

**Related:** `mfa.webauthn.register`, `mfa.webauthn.challenge`, `mfa.webauthn.verify`, `mfa.listFactors`

---

### `mfa.webauthn.register(params, overrides?)`

**Priority:** Medium | **Complexity:** Complex

**Signature:**

```ts
async mfa.webauthn.register(
  params: {
    friendlyName: string
    webauthn?: {
      rpId?: string        // Defaults to window.location.hostname
      rpOrigins?: string[] // Defaults to [window.location.origin]
      signal?: AbortSignal
    }
  },
  overrides?: Partial<PublicKeyCredentialCreationOptions>
): Promise<RequestResult<AuthMFAVerifyResponseData, WebAuthnError | AuthError>>
```

**Purpose:** Complete WebAuthn registration flow in a single call. Performs enroll + challenge + browser credential creation + verify. Registers a new WebAuthn factor and activates it.

**Parameters:**

- `params.friendlyName` (string, required) - Human-readable name for the credential
- `params.webauthn?.rpId?` (string) - Relying Party ID; defaults to `window.location.hostname`
- `params.webauthn?.rpOrigins?` (string[]) - Allowed origins; defaults to `[window.location.origin]`
- `params.webauthn?.signal?` (AbortSignal) - Optional abort signal
- `overrides?` (Partial\<PublicKeyCredentialCreationOptions\>) - Override `navigator.credentials.create()` options

**Returns:**

```ts
{
  data: {
    access_token: string
    token_type: 'bearer'
    expires_in: number
    refresh_token: string
    user: User
  },
  error: WebAuthnError | AuthError | null
}
```

**Usage:**

```ts
// Register a new security key
const { data, error } = await client.auth.mfa.webauthn.register({
  friendlyName: 'My YubiKey',
})
// Browser prompts user to create credential on security key
// On success, factor enrolled + verified, session upgraded to aal2

// With explicit RP config
const { data, error } = await client.auth.mfa.webauthn.register({
  friendlyName: 'Office Key',
  webauthn: {
    rpId: 'myapp.com',
    rpOrigins: ['https://myapp.com'],
  },
})

// With overrides for platform authenticator (e.g. Touch ID)
const { data, error } = await client.auth.mfa.webauthn.register(
  { friendlyName: 'Touch ID' },
  {
    authenticatorSelection: {
      authenticatorAttachment: 'platform',
      userVerification: 'required',
      residentKey: 'required',
    },
  }
)

// With abort signal
const controller = new AbortController()
const { data, error } = await client.auth.mfa.webauthn.register({
  friendlyName: 'My Key',
  webauthn: { signal: controller.signal },
})
// Cancel: controller.abort()
```

**Implementation Considerations:**

- **Backend Requirements:** Factor enrollment, WebAuthn challenge generation, credential verification, session token generation
- **Security:** Checks `browserSupportsWebAuthn()` first. Returns error if `rpId` is falsy. On enrollment failure, attempts cleanup: finds matching unverified WebAuthn factor by friendlyName and unenrolls it.
- **Complexity:** Complex - orchestrates full ceremony: browser check, enroll, challenge (server + browser credential.create()), verify. Error recovery with factor cleanup.
- **Dependencies:** Browser WebAuthn API, `navigator.credentials.create()`
- **Storage:** Session saved after successful verification

**Authentication Flow:**

```
mfa.webauthn.register({ friendlyName: 'My Key', webauthn: { rpId, rpOrigins, signal } })
  → Validate rpId exists (error if not)
  → Check browserSupportsWebAuthn() (error if not)
  → webauthn.enroll({ friendlyName }) → POST /factors → returns { id: factorId }
  → If enroll fails:
      → Cleanup: find matching unverified WebAuthn factor by friendlyName, unenroll it
      → Return error
  → webauthn.challenge({ factorId, friendlyName, webauthn: { rpId, rpOrigins }, signal }, { create: overrides })
      → POST /factors/{factorId}/challenge → server returns credential creation options
      → Sets user.name/displayName if blank (WebAuthn requires non-empty)
      → navigator.credentials.create({ publicKey, signal }) → user touches key
      → Returns { challengeId, webauthn: { type: 'create', credential_response } }
  → webauthn.verify({ factorId, challengeId, webauthn: { rpId, rpOrigins, type, credential_response } })
      → POST /factors/{factorId}/verify → server verifies + activates factor
      → Returns new session tokens (aal2)
  → Session saved, MFA_CHALLENGE_VERIFIED event
```

**GoTrue Backend Mapping:**

- Three sequential requests:
  1. `POST /factors` - enroll new WebAuthn factor
  2. `POST /factors/{factorId}/challenge` - get credential creation options
  3. `POST /factors/{factorId}/verify` - verify created credential

**Error Cases:**

- `AuthError('rpId is required for WebAuthn registration')` - rpId is falsy
- `AuthUnknownError('Browser does not support WebAuthn')` - missing WebAuthn APIs
- All `mfa.enroll` errors (step 1)
- All `mfa.webauthn.challenge` errors (step 2)
- All `mfa.webauthn.verify` errors (step 3)
- `WebAuthnError` - user cancelled, timeout, credential creation failed
- `AuthUnknownError('Unexpected error in register')` - catch-all

**Related:** `mfa.webauthn.authenticate`, `mfa.webauthn.enroll`, `mfa.webauthn.challenge`, `mfa.webauthn.verify`

---

## Summary

| Method | Type | Endpoint(s) | Notes |
|--------|------|-------------|-------|
| `mfa.enroll` | POST | `/factors` | Creates unverified factor (TOTP/phone/WebAuthn) |
| `mfa.webauthn.enroll` | POST | `/factors` | Wrapper: `enroll({ factorType: 'webauthn' })` |
| `mfa.challenge` | POST | `/factors/{id}/challenge` | Creates challenge; WebAuthn deserializes options |
| `mfa.webauthn.challenge` | POST | `/factors/{id}/challenge` + browser | Server challenge + `navigator.credentials.create/get` |
| `mfa.verify` | POST | `/factors/{id}/verify` | Verifies challenge; WebAuthn serializes credential |
| `mfa.webauthn.verify` | POST | `/factors/{id}/verify` | Wrapper for `verify()` with WebAuthn params |
| `mfa.challengeAndVerify` | POST | `/factors/{id}/challenge` + `/verify` | TOTP only: combined challenge+verify |
| `mfa.unenroll` | DELETE | `/factors/{id}` | Requires aal2 for verified factors |
| `mfa.listFactors` | GET | `/user` (via `getUser()`) | Categorizes user.factors by type/status |
| `mfa.getAuthenticatorAssuranceLevel` | - | None (JWT decode) | Client-side AAL/AMR inspection |
| `mfa.webauthn.authenticate` | POST | `/factors/{id}/challenge` + `/verify` + browser | Full auth ceremony for verified factors |
| `mfa.webauthn.register` | POST | `/factors` + `/factors/{id}/challenge` + `/verify` + browser | Full registration ceremony (enroll+challenge+verify) |

**AAL Levels:**
- `aal1` - conventional login only (password, OTP, magic link, social)
- `aal2` - conventional login + at least one MFA factor verified

**Factor Lifecycle:** `enroll()` (unverified) → `challenge()` → `verify()` (verified, session aal2)

**WebAuthn Ceremony Types:**
- `create` - for unverified factors (registration: `navigator.credentials.create`)
- `request` - for verified factors (authentication: `navigator.credentials.get`)
