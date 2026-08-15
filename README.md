# Glory Gym API — Developer Documentation

Detailed, per-section API guides for the **Dashboard (web) frontend** and the
**Mobile app** developers. Every endpoint here exists in the code and has been
verified against a running server — nothing is speculative.

- **Base URL (dev):** `http://localhost:3000` — no global prefix.
- **Swagger UI:** `GET /docs` (interactive, always in sync with the code).
- **Postman:** `postman/GloryGym.postman_collection.json` (master, all 207
  requests) + `postman/GloryGym-MobileApp.postman_collection.json` (mobile
  only) + `postman/GloryGym.postman_environment.json`.

## Sections

### `frontend/` — Dashboard (web) developers

| File | Covers |
| --- | --- |
| [frontend/01-auth.md](frontend/01-auth.md) | Employee login, tokens, roles & permissions model |
| [frontend/02-team-management.md](frontend/02-team-management.md) | Admins / Staff / Instructors screens, Add-Employee wizard, uploads, catalogs |
| [frontend/03-checkins.md](frontend/03-checkins.md) | Front-desk QR scanning for gym entry |
| [frontend/04-video-library.md](frontend/04-video-library.md) | Videos Library: grid/search, Add/Edit Video, Vimeo upload |
| [frontend/05-follow-up-programs.md](frontend/05-follow-up-programs.md) | Follow-Up Programs: list/stats/export, settings toggles, instructor rotation |
| [frontend/06-workouts.md](frontend/06-workouts.md) | Workouts: program templates + stages/videos, assigning to members — also backs the Instructor Dashboard's "Workouts" nav (list + 3-step wizard) |
| [frontend/07-gym-management.md](frontend/07-gym-management.md) | Gym Management: Products (image/category/export) + Push Notifications broadcast |
| [frontend/08-company-data.md](frontend/08-company-data.md) | Settings → Company Data: all 9 sidebar tabs — CMS pages (About/Vision/Mission/Goals/Terms/Privacy), Contact Methods (reorderable list), FAQ, Assessment Evaluation |
| [frontend/09-instructor-dashboard.md](frontend/09-instructor-dashboard.md) | Instructor's own Dashboard home: stat cards, reservations calendar, day-detail panel, Create Reservation, members-by-remaining-days |
| [frontend/10-coach-chat.md](frontend/10-coach-chat.md) | Coach Chat: assigning a member to an instructor (auto-creates the chat), the instructor dashboard's conversation list/history/read-state, Socket.IO send/receive |
| [frontend/11-members.md](frontend/11-members.md) | Members (Instructor Dashboard): list + stat cards, profile read, In-Body Test / Body Measurements CRUD |

### `mobile/` — Mobile app developers

| File | Covers |
| --- | --- |
| [mobile/01-auth.md](mobile/01-auth.md) | Signup with email OTP, login, forgot password, sessions |
| [mobile/02-profile.md](mobile/02-profile.md) | Settings, profile, change email/password, notifications, delete account, subscriptions, body records, avatar upload |
| [mobile/03-family.md](mobile/03-family.md) | Family members CRUD |
| [mobile/04-content.md](mobile/04-content.md) | About pages, FAQs, contact links, complaints & suggestions |
| [mobile/05-bookings-checkin.md](mobile/05-bookings-checkin.md) | Gym-entry QR check-in, bookings (cancel/check-in), post-session rating, notification inbox |
| [mobile/06-workouts.md](mobile/06-workouts.md) | My Exercises: list, exercise detail with stages/videos, logging weight |
| [mobile/07-onboarding.md](mobile/07-onboarding.md) | First-login questionnaire (PAR-Q + intake): status, submit, read back |
| [mobile/08-chat.md](mobile/08-chat.md) | Coach Chat: conversation list, message history, read state, Socket.IO send/receive |

---

## Conventions (read once — applies to every endpoint)

### 1. Response envelope

Every **successful** JSON response is wrapped:

```json
{ "success": true, "data": { /* payload */ } }
```

Paginated lists add `meta` at the top level:

```json
{
  "success": true,
  "data": [ /* items */ ],
  "meta": { "page": 1, "limit": 10, "total": 42, "totalPages": 5 }
}
```

Every **error** is wrapped:

```json
{ "success": false, "message": "Human readable message", "statusCode": 400, "errors": ["field-level messages (validation only)"] }
```

> Exception: the CSV export endpoints (`GET /{admins|staff|instructors}/export`)
> return a raw `text/csv` file download, not JSON.

### 2. Authentication header

All protected endpoints use a Bearer token:

```
Authorization: Bearer <accessToken>
```

There are **two completely separate token families**:

| Token family | Issued by | Works on |
| --- | --- | --- |
| Employee (dashboard) | `POST /auth/login` | `/auth/*`, `/admins`, `/staff`, `/instructors`, `/branches`, … |
| Member (mobile app) | `POST /mobile/auth/login` | `/mobile/*` only |

A member token on a dashboard endpoint → `401`, and vice versa. Never mix them.

### 3. Token lifecycle (both families)

- **Access token:** short-lived (default 15 min). Send on every request.
- **Refresh token:** long-lived (default 7 days), **single-use** — calling
  refresh **rotates** it (the old one is revoked, you get a new pair). Reusing
  an old refresh token → `401`.
- On any `401`: try one refresh; if refresh also fails → force re-login.
- Logout / password change / password reset / status change revoke refresh
  tokens server-side.

### 4. Pagination (all list endpoints)

Query params: `page` (1-based, default 1), `limit` (1–100, default 10),
`search`, `sortBy`, `sortOrder` (`asc`|`desc`, default `desc`).
`limit > 100` → `400`.

### 5. Validation errors

The global ValidationPipe returns **`400`** (not 422) with an `errors` array.
Unknown body fields are **rejected** (`"property X should not exist"`) — send
only documented fields.

### 6. Dates, money, language

- **Dates:** ISO-8601 strings. Send `"2003-09-19"` or full ISO; receive full
  ISO UTC (`"2003-09-19T00:00:00.000Z"`). Format for display client-side.
- **Money / decimals:** sent as **strings** (`"850.000"`). JSON responses
  normalize trailing zeros (`"850.500"` → `"850.5"`) — the value is exact;
  format client-side.
- **Bilingual fields** are `<field>En` / `<field>Ar` (e.g. `nameEn`/`nameAr`).
  Pick the one matching the app language; fall back to the other when null.

### 7. HTTP status cheat-sheet

| Code | Meaning here |
| --- | --- |
| 200 / 201 | OK / created |
| 400 | Validation failure or business rule (message says which) |
| 401 | Missing/invalid/expired token, wrong credentials, revoked refresh |
| 403 | Authenticated but not allowed (role/permission/status) |
| 404 | Not found (or not yours — ownership scoping returns 404, not 403) |
| 409 | Conflict (duplicate email/phone, entity still referenced, …) |
| 429 | OTP resend cooldown — retry after the seconds in the message |
| 503 | External service not configured/unavailable (SMTP / Cloudinary) |

---

## Client integration snippets

### Axios instance that unwraps the envelope + auto-refreshes (works for both families)

```ts
const api = axios.create({ baseURL: BASE_URL });

api.interceptors.request.use((cfg) => {
  const token = getAccessToken();
  if (token) cfg.headers.Authorization = `Bearer ${token}`;
  return cfg;
});

api.interceptors.response.use(
  (res) => res.data.data ?? res.data, // unwrap { success, data }
  async (err) => {
    const original = err.config;
    if (err.response?.status === 401 && !original._retried) {
      original._retried = true;
      try {
        // dashboard: /auth/refresh — mobile: /mobile/auth/refresh
        const r = await axios.post(`${BASE_URL}${REFRESH_PATH}`, {
          refreshToken: getRefreshToken(),
        });
        saveTokens(r.data.data); // rotation: BOTH tokens are new
        original.headers.Authorization = `Bearer ${r.data.data.accessToken}`;
        return api(original);
      } catch {
        logoutLocally(); // refresh dead -> back to login screen
      }
    }
    // err.response.data = { success:false, message, errors?, statusCode }
    return Promise.reject(err);
  },
);
```

### Showing API errors

Always display `error.response.data.message`; if `errors[]` exists (validation),
show those under the matching fields.
