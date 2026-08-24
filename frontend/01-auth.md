# 01 — Dashboard Auth (Employees)

> Audience: **web dashboard frontend**. Mobile devs: your auth is
> [../mobile/01-auth.md](../mobile/01-auth.md) — do **not** use these endpoints.

Login for employee accounts (`ADMIN` / `STAFF` / `INSTRUCTOR`). Employees are
created from Team Management ([02-team-management.md](02-team-management.md));
there is **no self-signup** for the dashboard.

Base path: `/auth`

| # | Method & path | Auth | Purpose |
| --- | --- | --- | --- |
| 1 | `POST /auth/login` | Public | Email + password → token pair + user |
| 2 | `POST /auth/refresh` | Public | Rotate refresh token → new pair |
| 3 | `POST /auth/logout` | Bearer | Revoke a refresh token |
| 4 | `GET /auth/me` | Bearer | Current user + permission keys |
| 5 | `GET /auth/me/profile` | Bearer | "Settings > Profile Management" — full self profile |
| 6 | `PATCH /auth/me/profile` | Bearer | "Edit Profile" |
| 7 | `PATCH /auth/me/password` | Bearer | "Update Password" |

---

## 1. `POST /auth/login`

**Body**

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `email` | string (email) | yes | Employee email |
| `password` | string | yes | min 6 |

**200**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "user": {
      "id": "cmq3...", "email": "admin@glorygym.com", "fullName": "Super Admin",
      "role": "ADMIN", "status": "ACTIVE",
      "departmentId": null, "primaryBranchId": null, "avatarUrl": null,
      "permissions": ["staff.read", "payroll.manage"]
    }
  }
}
```

**Errors**

| Code | When | UI action |
| --- | --- | --- |
| 401 | Wrong email or password (generic — no account-existence leak) | Show generic "invalid credentials" |
| 403 | Account is `INACTIVE` / `FREEZED` | Show "account disabled — contact admin" |

Store both tokens; route by `role`/`permissions` (below).

## 2. `POST /auth/refresh`

**Body:** `{ "refreshToken": "<jwt>" }`

**200:** same shape as login (new `accessToken` **and** new `refreshToken` —
rotation; persist both, discard the old ones).

**401:** invalid / expired / revoked / already-used token → force re-login.

## 3. `POST /auth/logout` (Bearer)

**Body:** `{ "refreshToken": "<jwt>" }` → `{ "message": "Logged out" }`.
Revokes that refresh token (only if it belongs to the caller). Access token
simply expires on its own (≤15 min) — also clear it locally.

## 4. `GET /auth/me` (Bearer)

Returns the same `user` object as login (fresh from DB — status/permission
changes apply immediately). Call it on app boot to restore the session.

## 5–7. Settings → "Profile Management" (self-service, own record only)

Any authenticated employee (`ADMIN`/`STAFF`/`INSTRUCTOR` — this isn't
role-specific) can view and edit **their own** record here — **no
`*.read`/`*.manage` permission required**, since you can always see and
edit yourself. This is deliberately separate from the Team Management
endpoints (`GET/PATCH /admins|staff|instructors/:id`), which manage *other*
employees and stay permission-gated.

### 5. `GET /auth/me/profile`

The same fully-hydrated shape `GET /{admins|staff|instructors}/:id`
returns (see [02-team-management.md](02-team-management.md)) — one call
covers all four Settings tabs at once: profile fields, job info
(department/reportingTo/branches/contract/salary), `documents[]`, and
`shifts[]` (the Attendance table) + your own granted `permissions[]` keys.
Filter/route to the right tab client-side; no separate call per tab.

### 6. `PATCH /auth/me/profile` — "Edit Profile"

Deliberately narrow — **only** personal-info fields, not job info:
`username`, `fullName`, `phoneCountryCode`, `phone`, `gender`,
`dateOfBirth`, `age`, `nationality`, `nationalId`, `city`, `streetName`,
`avatarUrl` (via `POST /uploads` first). Sending anything else (e.g.
`departmentId`, `baseSalary`, `permissions`) → `400`, whitelist-rejected —
those stay HR-controlled via the Team Management endpoints. `email` is
**not** self-editable either — there's no verified-change flow for
employees yet (unlike the mobile app's OTP-based email change); changing
it currently still requires an admin via Team Management.

### 7. `PATCH /auth/me/password` — "Update Password"

```json
{ "currentPassword": "OldPass1122#", "password": "NewPass1122#", "passwordConfirm": "NewPass1122#" }
```

Unlike Team Management's admin-driven `PATCH /{base}/:id/password` (which
resets *someone else's* password without knowing the old one), this one
**requires your current password** — the same convention as the mobile
app's own change-password flow. **400** if it's wrong or if
`password`/`passwordConfirm` don't match. **200** revokes all of your
refresh tokens (same as any password change) — you're logged out
everywhere and re-log in with the new password.

---

## Roles & permissions — what the frontend needs to know

- **`role`** is coarse: `ADMIN` | `STAFF` | `INSTRUCTOR`.
- **`permissions`** is an array of keys like `staff.read`, `branches.manage`.
- **ADMIN bypasses all permission checks** server-side — treat an ADMIN as
  having every permission in the UI too.
- Rule of thumb across the API: **GET endpoints need only a valid token**
  (some Team endpoints additionally need `<role>.read`); **write endpoints
  need a `*.manage` key**. Each section doc lists the exact key per endpoint.
- Drive menu/button visibility from `user.permissions`; the server enforces
  the same keys, so a hidden-but-called endpoint would return `403` anyway.

Permission catalog for the assignment UI: `GET /permissions` (see
[02-team-management.md](02-team-management.md) — returns keys grouped exactly
as the design's checkbox groups).

### Session behavior worth handling

- Access tokens are validated against a **fresh DB read** every request: if an
  admin deactivates a user or edits their permissions, it takes effect on the
  **next request**, not the next login. Expect sudden `401`/`403` and handle
  them gracefully (redirect to login / show "no access").
- Changing a user's password (via Team Management) revokes their refresh
  tokens → their session dies at the next refresh. **Your own** password
  change (§7) does the same thing — expect the current tab's next request
  to `401` too, not just other sessions.

⚠ **Documented UI note (Instructor Dashboard "Settings > Profile
Management" screens):** the profile card header in the source mockups
shows a *different* name/avatar/status/date ("Kristina Gislason", "Active",
"Oct 10, 2025") than the logged-in user shown in the top-right corner and
than the actual field values in every tab (all "Ahmed Hossam"'s own data)
— almost certainly a stale/reused header component copied from the Team
Management employee-profile screens (which legitimately show *another*
employee's name there), not evidence of a second person. `GET
/auth/me/profile` always returns **your own** record; render that header
from the response, not from the mockup's literal text.
