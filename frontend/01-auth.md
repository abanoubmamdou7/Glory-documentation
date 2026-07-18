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
  tokens → their session dies at the next refresh.
