# Sign-up: one permanent email sign-up

**Changed in Release 0 (2026-11).** The switchable review sign-up from
2026-10-19 is **retired**. It showed a separate sign-up screen only while the
gym had a server switch on, i.e. only during Apple's review. App Store
guideline 5.6 rejects exactly that, so it is gone for good.

## What the app does now

1. **Sign Up is always visible** on the login screen. No server call decides
   whether it exists.
2. It always uses the normal email sign-up:

   | Step | Endpoint | Notes |
   | --- | --- | --- |
   | 1 | `POST /mobile/auth/register` `{ fullName, phoneCountryCode, phone, email }` | `200`; a 6-digit code is emailed. `409` email/phone already registered; `503` email unavailable |
   | 2 | `POST /mobile/auth/verify-otp` `{ email, purpose: "REGISTRATION", code }` | `200 { otpToken }` |
   | 3 | `POST /mobile/auth/complete-registration` `{ otpToken, password, passwordConfirm }` | `201`, the member profile (no tokens) |
   | 4 | `POST /mobile/auth/login` `{ identifier, password }` | `200 { accessToken, refreshToken, member }` |

   Full details: [01-auth.md](01-auth.md).

## What to remove from the app

- **Do not call `GET /mobile/app-config` to decide anything.** It now always
  answers `{ "signupEnabled": false }` and will lose the field later.
- **Do not call `POST /mobile/auth/review-signup`.** It always answers `403`.
- Remove the review sign-up screen and any code that hides or shows sign-up
  based on a server value, the build number, the platform or the date.

## Accounts made by the old review sign-up

They are ordinary accounts and keep logging in with email or phone and
password. The gym can remove the ones nobody uses from the dashboard
(`DELETE /settings/mobile-app/review-accounts`), never while a build is in
review.
