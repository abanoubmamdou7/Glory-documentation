# 01 — Mobile Auth: sign-up, login, password recovery, guest, account deletion

> Audience: **Flutter app**. Final contract, branch `claude/release-a-lite`
> (not deployed). Base path `/mobile/auth` unless stated. Employee
> (dashboard) tokens are rejected everywhere under `/mobile/*` (401
> `TOKEN_INVALID`), and member tokens are rejected on the dashboard.

## Response envelope and error codes

```json
{ "success": true, "data": { … } }
{ "success": false, "statusCode": 400, "error": "OTP_EXPIRED", "message": "The code has expired — request a new one" }
```

- **Branch on `error`**, never on `message` (English, may be reworded).
- `statusCode` and `message` are exactly what they were before codes were
  added, so nothing that worked before breaks.
- Request-shape errors: `error: "VALIDATION_FAILED"`, `message: "Validation failed"`,
  plus `errors: string[]` (one line per broken rule; show a generic
  "check the form" or map field names yourself).
- Some codes carry extra fields: `attemptsRemaining`, `retryAfterSeconds`,
  `accountStatus` (listed per code below).

### All auth codes

| `error` | HTTP | Meaning | App action |
| --- | --- | --- | --- |
| `VALIDATION_FAILED` | 400 | body/query breaks a rule (`errors[]`) | fix the form |
| `EMAIL_TAKEN` | 409 | an account already uses this email | offer **Log in** and **Forgot password** |
| `PHONE_TAKEN` | 409 | an account already uses this phone | same |
| `PASSWORD_MISMATCH` | 400 | `password` ≠ `passwordConfirm` | show under the confirm field |
| `OTP_NOT_REQUESTED` | 400 | no live code for this email/purpose (never asked, already used, or replaced) | restart the flow / request a code |
| `OTP_INCORRECT` | 400 | wrong code; `attemptsRemaining` (0–4) | let them retry |
| `OTP_EXPIRED` | 400 | code older than its lifetime | show "Resend" |
| `OTP_ATTEMPTS_EXCEEDED` | 400 | 5 wrong tries, code dead | show "Resend" |
| `OTP_COOLDOWN` | 429 | asked for a new code too soon; `retryAfterSeconds` | disable the button for that many seconds |
| `OTP_PURPOSE_NOT_ALLOWED` | 400 | `verify-otp` called with `EMAIL_CHANGE`/`ACCOUNT_DELETE` | app bug |
| `OTP_TOKEN_INVALID` | 401 | `otpToken` invalid, expired (15 min), wrong purpose or already used | restart the flow |
| `EMAIL_DELIVERY_UNAVAILABLE` | 503 | the server cannot send email now; nothing was stored | "Try again later" |
| `INVALID_CREDENTIALS` | 401 | unknown identifier, wrong password, **or a gym-created account with no password yet** | show error + **Forgot password** |
| `ACCOUNT_DISABLED` | 403 on login, 401 elsewhere | the gym disabled the account; `accountStatus`: `INACTIVE` / `FREEZED` | "Contact the gym", sign out |
| `ACCOUNT_NOT_FOUND` | 404 | forgot-password: no account for this identifier | offer sign-up |
| `ACCOUNT_NO_EMAIL` | 400 | account has no email (forgot-password, deletion) | "Contact the gym" |
| `TOKEN_INVALID` | 401 | access token missing, malformed, expired, or an employee token | refresh once, else login |
| `REFRESH_TOKEN_INVALID` | 401 | refresh token invalid, expired, revoked or already used | go to login |
| `ACCOUNT_DELETED` | 401 | the account no longer exists | clear tokens, go to welcome |
| `CURRENT_PASSWORD_INCORRECT` | 400 | change-password: wrong current password | show under the field |
| `PASSWORD_NOT_SET` | 400 | change-password on an account with no password | use forgot-password |
| `EMAIL_UNCHANGED` | 400 | change-email to the same address | show error |
| `ACCOUNT_HAS_RECORDS` | 409 | deletion refused today (gym records); see deletion | "Contact the gym" (goes away with R0-2) |
| `REVIEW_SIGNUP_RETIRED` | 403 | the retired review sign-up was called | never call it |

Membership and Sandy codes (`MEMBERSHIP_REQUIRED` 403, `SANDY_QUOTA_EXCEEDED`
429) are in `20-membership-access.md`. **A 401 is never about membership.**

## The endpoints

| # | Method & path | Auth | Success |
| --- | --- | --- | --- |
| 1 | `POST /mobile/auth/register` | none | 200 |
| 2 | `POST /mobile/auth/verify-otp` | none | 200 |
| 3 | `POST /mobile/auth/resend-otp` | none | 200 |
| 4 | `POST /mobile/auth/complete-registration` | none | **201, no tokens** |
| 5 | `POST /mobile/auth/login` | none | 200, tokens |
| 6 | `POST /mobile/auth/forgot-password` | none | 200 |
| 7 | `POST /mobile/auth/reset-password` | none | 200 |
| 8 | `POST /mobile/auth/refresh` | refresh token in body | 200, new tokens |
| 9 | `POST /mobile/auth/logout` | Bearer | 200 |
| 10 | `GET /mobile/auth/me` | Bearer | 200 |
| 11 | `POST /mobile/profile/delete-account/request` | Bearer | 200 |
| 12 | `DELETE /mobile/profile/delete-account/confirm` | Bearer | 200 |
| — | `POST /mobile/auth/review-signup` | — | always 403 `REVIEW_SIGNUP_RETIRED`, never call it |
| — | `GET /mobile/app-config` | none | always `{ "signupEnabled": false }`, ignore it |

**`complete-registration` does NOT return tokens.** After 201 the app calls
`login` with the same email and password.

## Codes sent by email (both flows)

- 6 digits, email only. Valid `expiresInSeconds` (production 60 s); resend
  allowed after `resendCooldownSeconds` (60 s). Use the response values.
- A new code (resend, or calling register/forgot again) kills the old one.
- 5 wrong tries kill the code (`OTP_ATTEMPTS_EXCEEDED`).
- `verify-otp` success gives `otpToken` (15 min, one use) for exactly one
  `complete-registration` or `reset-password`.

## Sign-up

```
register ─200─▶ verify-otp ─200 {otpToken}─▶ complete-registration ─201─▶ login ─200 {tokens}
                    └── resend-otp (after the cooldown)
```

### 1. `POST /mobile/auth/register`

```json
{ "fullName": "أحمد حسام", "phoneCountryCode": "+962", "phone": "791234567", "email": "a@x.com" }
```

| Field | Rule |
| --- | --- |
| `fullName` | 1–120 chars, trimmed (stored as the full name; it is not a login username) |
| `phoneCountryCode` | `^\+\d{1,4}$` |
| `phone` | `^\d{6,15}$`, without the country code |
| `email` | valid email; trimmed and lower-cased by the server |

`200` → `{ "email", "purpose": "REGISTRATION", "expiresInSeconds", "resendCooldownSeconds" }`.
Nothing is created yet. Errors: `VALIDATION_FAILED`, `EMAIL_TAKEN`,
`PHONE_TAKEN`, `OTP_COOLDOWN`, `EMAIL_DELIVERY_UNAVAILABLE`.

### 2. `POST /mobile/auth/verify-otp`

`{ "email", "purpose": "REGISTRATION" | "PASSWORD_RESET", "code": "123456" }`
→ `200 { "otpToken", "purpose" }`. Errors: `VALIDATION_FAILED`,
`OTP_NOT_REQUESTED`, `OTP_INCORRECT`, `OTP_EXPIRED`, `OTP_ATTEMPTS_EXCEEDED`,
`OTP_PURPOSE_NOT_ALLOWED`.

### 3. `POST /mobile/auth/resend-otp`

`{ "email", "purpose" }` → `200`, same shape as register. Errors:
`OTP_NOT_REQUESTED`, `OTP_COOLDOWN`, `EMAIL_DELIVERY_UNAVAILABLE`.

### 4. `POST /mobile/auth/complete-registration`

`{ "otpToken", "password", "passwordConfirm" }`. Password: ≥ 8 chars, at
least one digit and one letter.

`201` → the profile, **no tokens**:

```json
{ "id": "cm…", "memberCode": "10421", "fullName": "أحمد حسام", "email": "a@x.com",
  "phone": "791234567", "phoneCountryCode": "+962", "avatarUrl": null,
  "status": "ACTIVE", "source": "APP", "emailVerifiedAt": "…",
  "onboardingCompleted": false, "createdAt": "…" }
```

Errors: `VALIDATION_FAILED` (weak password), `PASSWORD_MISMATCH`,
`OTP_TOKEN_INVALID`, `EMAIL_TAKEN` / `PHONE_TAKEN` (someone finished first).

## Login and sessions

### 5. `POST /mobile/auth/login`

`{ "identifier", "password" }`. One field, four kinds of value (trimmed and
lower-cased; **don't validate it as an email on the device**):

| Value | Looked up as |
| --- | --- |
| contains `@` | email |
| all digits | **member code** (number on the card), then phone |
| other, no `@` | **username** set by the gym (case-insensitive), then phone |

`200` → `{ "accessToken", "refreshToken", "member": <profile> }`. Errors:
`VALIDATION_FAILED`, `INVALID_CREDENTIALS`, `ACCOUNT_DISABLED` (403,
`accountStatus`). A member with an expired membership **still logs in**.

Tokens: access 15 min, refresh 7 days. Header `Authorization: Bearer <accessToken>`.

### 8. `POST /mobile/auth/refresh`

`{ "refreshToken" }` → `200` new pair. The sent token is now dead; store the
new one first. Errors: `REFRESH_TOKEN_INVALID`, `ACCOUNT_DISABLED` (401),
`ACCOUNT_DELETED`.

### 9. `POST /mobile/auth/logout`

Bearer + `{ "refreshToken" }` → `200`. Always succeeds.

### 10. `GET /mobile/auth/me`

Bearer → `200` profile. Errors (any member endpoint): `TOKEN_INVALID`,
`ACCOUNT_DISABLED` (401), `ACCOUNT_DELETED`.

**Rule for any 401:** `TOKEN_INVALID` → refresh once and retry;
`REFRESH_TOKEN_INVALID` → login; `ACCOUNT_DISABLED` / `ACCOUNT_DELETED` →
sign out with the message.

## Password recovery

```
forgot-password ─200─▶ verify-otp {PASSWORD_RESET} ─200 {otpToken}─▶ reset-password ─200─▶ login
```

- **6. `POST /mobile/auth/forgot-password`** `{ "identifier" }` (same four
  kinds as login) → `200 { email, purpose: "PASSWORD_RESET", … }`; the code
  goes to the account's email. Errors: `ACCOUNT_NOT_FOUND`, `ACCOUNT_NO_EMAIL`,
  `OTP_COOLDOWN`, `EMAIL_DELIVERY_UNAVAILABLE`.
- **7. `POST /mobile/auth/reset-password`** `{ otpToken, password, passwordConfirm }`
  → `200 { "message": "Password reset successfully" }`; **every session is
  signed out**. Errors: `VALIDATION_FAILED`, `PASSWORD_MISMATCH`, `OTP_TOKEN_INVALID`.

A member the gym created at the desk has no password: `register` answers
`EMAIL_TAKEN`/`PHONE_TAKEN` and `login` answers `INVALID_CREDENTIALS`.
**Forgot password is how they set one.**

## Guest access (no account)

No guest login or token. Without a token only these answer; every other
`/mobile` route answers 401 `TOKEN_INVALID`:

| Route | Content |
| --- | --- |
| `GET /mobile/app-config` | `{ signupEnabled: false }` (ignore) |
| `GET /mobile/content/pages`, `/mobile/content/pages/:key` | about, vision, terms, privacy … |
| `GET /mobile/content/faqs` | FAQs |
| `GET /mobile/content/contact` | contact methods |
| `GET /mobile/videos`, `/mobile/videos/categories`, `/mobile/videos/:id` | public video library |

## Account deletion (current behaviour; R0-2 not built)

```
delete-account/request ─200─▶ (code by email) ─▶ delete-account/confirm {code} ─200─▶ clear tokens → welcome
```

- **11. `POST /mobile/profile/delete-account/request`** — Bearer, no body →
  `200 { email, purpose: "ACCOUNT_DELETE", expiresInSeconds, resendCooldownSeconds }`.
  Errors: `ACCOUNT_NO_EMAIL`, `OTP_COOLDOWN`, `EMAIL_DELIVERY_UNAVAILABLE`.
- **12. `DELETE /mobile/profile/delete-account/confirm`** — Bearer +
  `{ "code" }` → `200 { "message": "Account deleted" }`. Errors:
  `OTP_NOT_REQUESTED`, `OTP_INCORRECT`, `OTP_EXPIRED`, `OTP_ATTEMPTS_EXCEEDED`,
  `ACCOUNT_HAS_RECORDS` (**today only**), `TOKEN_INVALID`.
- A code works once, even when the answer is `ACCOUNT_HAS_RECORDS`; request
  a new one to retry (the retry with the old code answers `OTP_NOT_REQUESTED`).
- The endpoints and shapes stay the same after R0-2; `ACCOUNT_HAS_RECORDS`
  disappears.

## Profile flows that share these codes

- `POST /mobile/profile/change-password` `{ currentPassword, password, passwordConfirm }`:
  `PASSWORD_MISMATCH`, `PASSWORD_NOT_SET`, `CURRENT_PASSWORD_INCORRECT`; success
  signs out every session.
- `POST /mobile/profile/change-email/request` `{ newEmail }`: `EMAIL_UNCHANGED`,
  `EMAIL_TAKEN`, `OTP_COOLDOWN`, `EMAIL_DELIVERY_UNAVAILABLE`.
- `POST /mobile/profile/change-email/confirm` `{ code }`: the `OTP_*` codes,
  `EMAIL_TAKEN`.

## Limits that exist today

Per email address: one code per 60 s, 5 attempts per code. No per-device or
per-IP limit on `register`.
