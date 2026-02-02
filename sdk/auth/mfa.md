# Multi-Factor Authentication (MFA)

**Priority:** MEDIUM (deferred)
**Methods:** 12
**Status:** Not Started

Multi-factor authentication including TOTP, SMS, and WebAuthn.

---

## Enrollment (2 methods)

### mfa.enroll
### mfa.webauthn.enroll

---

## Challenges (2 methods)

### mfa.challenge
### mfa.webauthn.challenge

---

## Verification (3 methods)

### mfa.verify
### mfa.webauthn.verify
### mfa.challengeAndVerify

---

## Management (3 methods)

### mfa.unenroll
### mfa.listFactors
### mfa.getAuthenticatorAssuranceLevel

---

## WebAuthn (2 methods)

### mfa.webauthn.authenticate
### mfa.webauthn.register
