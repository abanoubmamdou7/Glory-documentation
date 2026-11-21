# Glory Gym API — Developer Documentation

Detailed, per-section API guides for the **Dashboard (web) frontend** and the
**Mobile app** developers. Every endpoint here exists in the code and has been
verified against a running server — nothing is speculative.

- **Base URL (dev):** `http://localhost:3000` — no global prefix.
- **Swagger UI:** `GET /docs` (interactive, always in sync with the code).
- **Postman:** `postman/GloryGym.postman_collection.json` (master, all 502
  requests) + `postman/GloryGym-MobileApp.postman_collection.json` (mobile
  only) + `postman/GloryGym-DashboardGaps.postman_collection.json` (this
  session's additions — Admin Dashboard, Reports, Targets, Partnerships,
  Scheduling, Check-in history, Settings, Instructor Reports, and the
  Push/Follow-up/Reservation lifecycle actions) +
  `postman/GloryGym.postman_environment.json`.

> **Looking for everything at once?** [`docs/README.md`](../docs/README.md) is
> the project-wide index: the API reference plus every guide below, in one list.

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
| [frontend/12-instructor-workouts.md](frontend/12-instructor-workouts.md) | Instructor-only view of the Workouts screens (same API as 06, instructor framing) |
| [frontend/13-instructor-follow-up-programs.md](frontend/13-instructor-follow-up-programs.md) | Instructor-only view of Follow-Up Programs (same API as 05, instructor framing) |
| [frontend/14-instructor-settings.md](frontend/14-instructor-settings.md) | Settings → Profile Management: an employee viewing/editing their **own** account (`/auth/me/*`) |
| [frontend/15-sandy-ai.md](frontend/15-sandy-ai.md) | Sandy AI: the knowledge base behind the assistant (paste/import/reindex), retrieval preview, member-conversation oversight |
| [frontend/16-admin-assistant.md](frontend/16-admin-assistant.md) | Admin Assistant: ask the database in plain language (gated by `assistant.use`, permission-scoped, read-only) — chat, conversation history + SQL audit, readable-schema hint |
| [frontend/16a-admin-assistant-permissions.md](frontend/16a-admin-assistant-permissions.md) | Admin Assistant permission-awareness (2026-09-12): the `assistant.use` gate, the permission→data-area scope map (accountant ≠ HR), the new `PERMISSION_SCOPE` refusal, per-caller schema, medical-records block |
| [frontend/17-dashboard-gaps.md](frontend/17-dashboard-gaps.md) | Index for the "missing APIs" closure pass: unified Admin Dashboard, Reports & Analytics, Targets & Performance, Partnerships, Scheduling & Classes, Check-in history, Settings (notifications + email templates), Instructor Reports, and the new Push/Follow-up/Reservation lifecycle actions |
| [frontend/18-inbody-integration.md](frontend/18-inbody-integration.md) | InBody device integration (LookinBody Web): In-Body tests land on the member profile automatically — setup checklist, webhook, events log + retry, per-member link/register/backfill, field mapping to verify |
| [frontend/19-onboarding-questions.md](frontend/19-onboarding-questions.md) | Settings: dashboard-configurable catalog behind the mobile app's first-login intake wizard |
| [frontend/20-reports-analytics.md](frontend/20-reports-analytics.md) | Reports & Analytics tables: every Figma report screen (Sales, Members/Memberships, Booking, Account Balance, Staff attendance, Transaction ledger, Canceled Memberships, Accounting Entry, Follow-up tracking, Payroll, Salary + Add Salary Details, Month Target, Assessment Evaluation) → its cards/list/export endpoints, definitions and gotchas |
| [frontend/21-named-screen-permissions.md](frontend/21-named-screen-permissions.md) | Named screen permission keys (2026-09-19): one key per sidebar screen, the label-is-the-contract rule, `permissionLabels` on /auth/me, what is published-but-not-enforced, group-name deltas, the `grantAll` gotcha |
| [frontend/22-commercial-core.md](frontend/22-commercial-core.md) | Commercial core (2026-09-24): multi-beneficiary `POST /sales/checkout` + `Idempotency-Key`, the new `PARTIALLY_PAID` invoice status, households/guardians and the shared-phone rule, the stored outstanding-balance check-in policy (ALLOW_WARN/GRACE/BLOCK/MANAGER_OVERRIDE) and the permissioned, audited manager override |
| [frontend/23-missing-live-routes-2026-09-27.md](frontend/23-missing-live-routes-2026-09-27.md) | The 2026-09-27 missing-routes list: subscription unfreeze (and the days it gives back), check-in check-out (vs. reversal), Sandy's monthly coaching plans (draft → coach review → approve into a real workout), the coaching cycle object, and typed onboarding answers |
| [frontend/24-two-coaches.md](frontend/24-two-coaches.md) | Two coaches per member (2026-09-29): the gym slot vs the personal-training slot, what a PT sale assigns automatically, `coaches` / `ptInstructor` / `currentPt` on the member reads, `coachTypes` on both chat lists, and the backfill for existing PT clients |
| [frontend/25-freeze-lifecycle.md](frontend/25-freeze-lifecycle.md) | Freeze lifecycle (2026-09-30): the three different buttons on a frozen sale (freeze / unfreeze / **cancel-freeze**) and what each does to `endDate`, the package quota and the timeline, the two-step refund on a paid freeze fee, and the job that resumes a sale by itself once its freeze window ends |
| [frontend/26-sandy-advisor-and-nutrition-review.md](frontend/26-sandy-advisor-and-nutrition-review.md) | Sandy as advisor (2026-10-03): the insights screen (financial analysis, forecast, evidence-backed recommendations, the accept/dismiss decision board, `WRITE_REFUSED` in chat) + the coach's nutrition-plan review queue (approve / edit & approve / reject, safety-rule errors, statuses) |
| [frontend/27-sandy-round3.md](frontend/27-sandy-round3.md) | Sandy round 3 (2026-10-03): AI usage & budget screen, proactive member-message templates (approve → live, results), the what-if simulator, recommendation impact on the decision board, and the coach copilot (morning summary + "suggest a reply") |
| [frontend/28-sandy-confirmed-actions.md](frontend/28-sandy-confirmed-actions.md) | Sandy executes only a confirmed action (2026-10-03): the action card in the chat, the Execute button on recommendations (⚠ `recommendation.action` is now an object), the preview/confirm dialog with Idempotency-Key, the error codes, and the audit screen |
| [frontend/29-notifications-and-push.md](frontend/29-notifications-and-push.md) | Notifications & device push (2026-10-03): the inbox-first delivery model, the mobile/staff device-registration checklist, the published 50-event catalog (`GET /notifications/events`), the reminder sweep and its dedupe keys, diagnosing under-delivery with `reach.coveragePct`, and what is deliberately never put in a notification |
| [frontend/30-coach-session-check-in.md](frontend/30-coach-session-check-in.md) | Marking a PT session attended (2026-10-15): the coach's manual check-in next to the QR and in-app routes, the once-per-session rule and how the UI should treat a 409, every error, and the two limits (one appointment a day, no coach-side undo) |

### `mobile/` — Mobile app developers

| File | Covers |
| --- | --- |
| [mobile/01-auth.md](mobile/01-auth.md) | Final auth contract: sign-up (complete-registration returns no tokens → login), login by email / phone / member code / username, password recovery, sessions, guest access, account deletion, and the stable `error` code of every failure |
| [mobile/02-profile.md](mobile/02-profile.md) | Settings, profile, change email/password, notifications, delete account, subscriptions, body records, avatar upload |
| [mobile/03-family.md](mobile/03-family.md) | Family members CRUD |
| [mobile/04-content.md](mobile/04-content.md) | About pages, FAQs, contact links, complaints & suggestions |
| [mobile/05-bookings-checkin.md](mobile/05-bookings-checkin.md) | Gym-entry QR check-in, bookings (cancel/check-in), post-session rating, notification inbox |
| [mobile/06-workouts.md](mobile/06-workouts.md) | My Exercises: list, exercise detail with stages/videos, logging weight |
| [mobile/07-onboarding.md](mobile/07-onboarding.md) | First-login intake wizard — dashboard-configurable, dynamic questions rendered generically by type |
| [mobile/08-chat.md](mobile/08-chat.md) | Coach Chat: conversation list, message history, read state, Socket.IO send/receive |
| [mobile/09-sandy-ai.md](mobile/09-sandy-ai.md) | Sandy AI: asking the gym assistant, conversations, citations, and the refusal states (which are `200`, not errors) |
| [mobile/10-sandy-medical-documents.md](mobile/10-sandy-medical-documents.md) | Sandy reads medical files & X-rays (vision), warns + refers to a doctor, and stores a private per-member memory (2026-09-08 update) |
| [mobile/12-inbody-history.md](mobile/12-inbody-history.md) | InBody history: every In-Body test (device + staff-entered) with change-since-last-test, summary card and chart series |
| [mobile/13-private-sessions.md](mobile/13-private-sessions.md) | Private sessions (PT) end to end: the booking cards, **scanning the coach's QR to check in** (and how it differs from the gym-entry QR, which points the other way), every scan error with copy, the rating step and picking the PT chat thread |
| [mobile/14-nutrition-plans.md](mobile/14-nutrition-plans.md) | Nutrition plans after every InBody test (2026-10-03): how an approved plan arrives (a Sandy conversation + inbox notification), the optional "My nutrition plan" screen endpoints and shape, goal labels, empty states |
| [mobile/15-sandy-proactive-messages.md](mobile/15-sandy-proactive-messages.md) | Sandy messages first (2026-10-03): how proactive messages arrive (a "💚 ساندي" conversation + inbox) and the member's mute switch |
| [mobile/16-notification-settings.md](mobile/16-notification-settings.md) | Notification settings (2026-10-15): the per-type mute screen built off the server's own catalog, why a muted type still shows in the inbox, the pushEnabled master switch, the two types that cannot be muted, and gym-wide quiet hours |
| [mobile/17-public-videos.md](mobile/17-public-videos.md) | Public video library (2026-10-19): the three no-token endpoints for a browse screen, every nullable field and what to render instead, why you see fewer videos than staff do, and why a 404 means "retired" rather than "broken" |
| [mobile/18-signup-screen-toggle.md](mobile/18-signup-screen-toggle.md) | Sign-up: one permanent email sign-up (register → verify-otp → complete-registration). The review sign-up and its server switch are retired in Release 0 (App Store 5.6): what the app must remove, and what happens to accounts it created |
| [mobile/19-group-classes.md](mobile/19-group-classes.md) | Group classes timetable (2026-11-16): `GET /mobile/group-classes`, Amman-time fields, branch/package scoping, cancelled classes included |
| [mobile/20-membership-access.md](mobile/20-membership-access.md) | Membership access (Release A-lite): `GET /mobile/membership` (state, canUse, Sandy quota), 403 MEMBERSHIP_REQUIRED on gym services, first-visit activation, 429 SANDY_QUOTA_EXCEEDED with the Amman-midnight reset, and the authorization matrix for every mobile route |
| [mobile/11-sandy-mobile-update-2026-09-10.md](mobile/11-sandy-mobile-update-2026-09-10.md) | **Mobile dev handoff (2026-09-10):** Sandy streaming (SSE) endpoint + cleaner plain-text/emoji formatting — what to change in the app (with Flutter sample) |

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
