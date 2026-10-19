# The sign-up screen (and the switch that hides it)

**New 2026-10-19.** Apple will not approve an app a reviewer cannot get into,
and Glory creates its members at the front desk — so the app only ever offered a
login form for credentials a reviewer doesn't have.

This adds a **self-service sign-up** that needs no OTP, no e-mail and no staff
action: one request and the reviewer is inside the app. It sits behind a
**server-side switch**, so once the app is approved the gym turns it off and the
screen disappears — **no new build, no resubmission**.

## The contract, in two endpoints

| | |
| --- | --- |
| `GET /mobile/app-config` | public. Call on launch, **before** the auth screens |
| `POST /mobile/auth/review-signup` | public. Creates the account and logs them in |

### 1. Ask whether the screen exists

```
GET /mobile/app-config          // no Authorization header
→ { "success": true, "data": { "signupEnabled": true } }
```

- `signupEnabled: true` → show the sign-up screen (and any "Create account"
  link that leads to it).
- `signupEnabled: false` → **the screen does not exist.** Hide the button, hide
  the route, don't deep-link to it. The app is back to login-only, exactly as it
  was before this feature.

⚠ **Treat anything that isn't an explicit `true` as `false`** — a network error,
a timeout, a malformed body. Failing closed means a hiccup hides a screen for a
moment; failing open means a public sign-up form appears in a shipped app that
isn't supposed to have one.

Call it on **every launch** (and on returning from background, if that's cheap).
It's a tiny unauthenticated GET. Don't cache the result across launches: the
whole point is that the gym can flip it without you shipping anything.

### 2. The sign-up itself

```
POST /mobile/auth/review-signup
{
  "fullName": "App Review",
  "email": "review@example.com",
  "password": "Review1234",
  "passwordConfirm": "Review1234",
  "phone": "1023359621",          // optional
  "phoneCountryCode": "+962"      // optional, only with phone
}
```

**`201` returns exactly what login returns:**

```json
{ "accessToken": "…", "refreshToken": "…", "member": { "id": "…", "memberCode": "50006", "fullName": "App Review", … } }
```

So: **store the tokens and go straight to the home screen.** No OTP screen, no
"check your email", no second form. That single hop is the entire reason this
exists — a reviewer who has to read an inbox is a reviewer who gives up.

Password rules are the same as the normal flow: **at least 8 characters, at
least one letter, at least one digit**, and `passwordConfirm` must match.
Validate client-side too, so the reviewer sees the rule instead of a 400.

### Errors

| Status | Why | What to show |
| --- | --- | --- |
| `400` | password too short / no digit / no letter / mismatch / bad e-mail | the specific field error from `message` |
| `409` | that e-mail (or phone) already has an account | "You already have an account — log in instead", and offer the login screen |
| `403` | **the switch is off**, or the gym's account cap is full | re-fetch `/mobile/app-config` and hide the screen; don't retry |
| `429` | too many accounts created from this device in the last hour | "Try again later" |

⚠ **`403` is the important one.** It's what a build with a stale config gets
after the gym turns the switch off, and it's deliberate: an old app can't keep
creating accounts. On `403`, re-read the config and take the screen away.

## What the account actually is

A **real member**: a real row, a real member code, a real password. The reviewer
can log out and log back in through the ordinary
`POST /mobile/auth/login` (by e-mail, phone, or that member code) and everything
behaves normally.

Two honest consequences to design around:

1. **It has no membership.** Nothing was sold, so the subscriptions screen,
   bookings, workouts and InBody history are all legitimately empty. Make the
   empty states look deliberate — "No active membership yet · Visit the gym to
   subscribe" — not like a failed request. A reviewer who sees a polished empty
   state reads it as a feature; one who sees a spinner or a blank page reads it
   as a bug. **This is the single most important screen in this whole document.**
   If a reviewer rejects the app for looking empty, ask the gym to sell that
   account a complimentary package from the dashboard — then every screen fills
   in with real data and nothing in the app has to change.
2. **The e-mail is unverified** (nobody proved they own it). Forgot-password
   still works the normal way — it sends a code to that address — so a reviewer
   using a fake address can't recover the account. Tell them to keep the password.

## For the gym (so you can answer "how do I turn it off?")

The dashboard, under Settings:

- `GET /settings/mobile-app` — the switch plus `reviewAccountCount`, how many of
  these accounts exist right now.
- `PATCH /settings/mobile-app` `{ "reviewSignupEnabled": false }` — off. Takes
  effect immediately: the next `/mobile/app-config` says `false` and the sign-up
  endpoint starts refusing.
- `DELETE /settings/mobile-app/review-accounts` — the tidy-up after approval.
  Removes the accounts the sign-up created, and **skips any that were sold a real
  membership** (if staff gave a reviewer a real subscription, that's a real
  member now).

Both need the `settings.manage` permission (`settings.view` to read).

**The switch is OFF by default**, including on a fresh install — so deploying
this doesn't open public sign-up by itself. Someone has to turn it on.

## The normal signup flow is untouched

`POST /mobile/auth/register` → `verify-otp` → `complete-registration` (with the
e-mail OTP) still exists and still works exactly as documented in
[01-auth.md](01-auth.md). This is a separate, switchable door — not a replacement.
If the gym ever wants genuine public self-registration, that OTP flow is the one
to put behind the button, and this one can be retired.
