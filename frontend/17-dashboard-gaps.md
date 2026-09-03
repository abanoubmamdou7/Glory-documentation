# 17 — Dashboard Gaps: Admin Dashboard, Reports, Targets, Partnerships, Scheduling, Check-in History, Settings, Instructor Reports

> Audience: **web dashboard frontend**. This is a consolidated index for
> everything built to close the "Missing APIs" contract
> (`glory-gym-missing-apis.postman_collection.json`) in one pass. Each
> feature's full endpoint reference lives in `docs/API.md` (linked below);
> this page is the map, plus the decisions/gotchas that don't fit a
> reference doc.

## What was built

| Contract section | Real endpoints (this session) | `docs/API.md` section |
| --- | --- | --- |
| Admin Dashboard (unified) | `GET /admin-dashboard/overview`, export create/status/download | [Admin Dashboard](../../docs/API.md#admin-dashboard-admin-dashboard) |
| Check-in History & Manual Check-in | `GET/POST /checkins`, `GET/DELETE /checkins/:id`, `GET /checkins/export` | [Check-ins (staff — history, manual entry, reversal)](../../docs/API.md#check-ins-staff--history-manual-entry-reversal) |
| Partnerships | Full CRUD + activate/deactivate + export, under `/partnerships` | [Partnerships](../../docs/API.md#partnerships-partnerships) (nested in "Sales domain") |
| Scheduling & Classes | Full CRUD + cancel + recurrence + export, under `/schedule-events` | [Scheduling & Classes](../../docs/API.md#scheduling--classes-schedule-events) |
| Reports & Analytics | 6 report endpoints + async export, under `/reports` | [Reports & Analytics](../../docs/API.md#reports--analytics-reports) |
| Targets & Performance | Full plan/assignment/adjustment/payout lifecycle, under `/targets` | [Targets & Performance](../../docs/API.md#targets--performance-targets) |
| Notification Settings | `GET/PATCH /settings/notifications`, `GET/PATCH /instructor-settings/notifications` | [Settings — Notifications & Email Templates](../../docs/API.md#settings--notifications--email-templates) |
| Email Templates | Full CRUD + preview + test-send, under `/email-templates` | same section as above |
| Instructor Settings | Covered by the existing `/auth/me/profile` (see doc 14) + the notification settings above | [14-instructor-settings.md](14-instructor-settings.md) |
| Push Notifications — edit/delete/send/cancel | `PATCH`/`DELETE`/`POST .../send`/`POST .../cancel` on `/push-notifications/:id` | [Push Notifications (Dashboard)](../../docs/API.md#push-notifications-dashboard) |
| Follow-up Programs — edit/delete/complete/reopen | same shape on `/follow-up-programs/:id` | [Follow-Up Programs](../../docs/API.md#follow-up-programs) |
| Instructor Reservations — edit/cancel | same shape on `/instructor-dashboard/reservations/:id` | [Instructor Dashboard](../../docs/API.md#instructor-dashboard) |
| Instructor Reports | 3 report endpoints + export, under `/instructor-reports` | [Instructor Reports](../../docs/API.md#instructor-reports-instructor-reports) |

Every one of these is backed by real, verified code — none of the
contract's "proposed backend endpoint" fallback language applies anymore.

## New shared building blocks (backend-internal, mentioned for context)

- **`ExportsService`** (`src/modules/exports/`) — one engine behind every
  `POST .../exports` in the app now: renders `CSV`/`XLSX`/`PDF` from a
  generic `{title, summary?, sections[]}` shape, persists an `ExportJob`
  row, optionally emails the file, and serves requester-scoped
  status/download. This is why every export family (dashboard, reports,
  targets, instructor-reports) behaves identically — same job states
  (`PENDING`→`PROCESSING`→`COMPLETED`/`FAILED`), same `downloadUrl`
  pattern, same `EMAIL` delivery rule.
- **`NotifyService`** (`src/modules/notify/`) — the shared "create an
  in-app `AUTO` push notification for specific members" helper. Powers the
  schedule-cancel and reservation-cancel "notify the affected member(s)"
  behavior; the same rows the mobile `GET /mobile/notifications` reads.
- A shared **date-range resolver** (`resolveRange`) standardizes
  `dateFrom`/`dateTo`/`timezone` handling (default: first day of the
  current month → today, `Africa/Cairo`) across every report/dashboard
  endpoint — so all of them share the exact same "what does an omitted
  range mean" behavior.

## Field-name mapping vs. the original contract

The contract (`glory-gym-missing-apis.postman_collection.json`) used
idealized field names in a few places; the real implementation follows
this project's existing conventions instead (documented per-endpoint in
`docs/API.md`, summarized here for a quick scan):

| Contract name | Actual field |
| --- | --- |
| Push `messageEn`/`messageAr` | `bodyEn`/`bodyAr` |
| Push `audienceType` | `targetAudience` |
| Follow-up `followUpType` | `recordType` (no `ASSESSMENT` value — enum is `WORKOUT`\|`BODY_MEASUREMENTS`\|`IN_BODY_TEST`\|`STUDENT`) |
| Follow-up `followUpId` | the program's own `id` |
| Reservation `startAt` | `scheduledAt` |
| Reservation `type: PERSONAL_TRAINING` | `PT_SESSION` (enum is `ASSESSMENT`\|`PT_SESSION`\|`PACKAGE`) |
| Instructor Reports `reportType` naming | `OVERVIEW`\|`MEMBERS`\|`ATTENDANCE` (not the contract's report names) |

## Gotchas checklist

- **Manual check-in is a staff override, not a stricter QR clone.** Unlike
  `POST /checkin/scan`, `POST /checkins` does **not** refuse a member with
  no active subscription — it records the visit and returns `warnings`
  instead, so front desk can decide.
- **A check-in "delete" is a reversal, not a hard delete.** The row stays
  (audit trail); lists/export hide it by default (`includeReversed=true`
  to see it).
- **Scheduling & Classes recurrence expands eagerly.** Creating a weekly
  class for a semester creates every individual `ScheduleEvent` row
  up-front (capped at 200 occurrences, `400` beyond that), sharing one
  `seriesId`. Editing/cancelling is per-occurrence — there's no
  "edit the whole series" endpoint yet.
- **`GET /admin-dashboard/overview` vs. `GET /reports/overview`** are
  deliberately different: the dashboard one is the "everything at a
  glance, including alerts and a live activity feed" home-screen shape;
  the reports one is a leaner, export-friendly KPI/series shape meant for
  the dedicated Reports screen. Don't conflate them.
- **Targets scoring is entirely live-computed** except at the three points
  it's persisted as an audit trail (`recalculate`, `lock`, `close` write
  `TargetSnapshot` rows). `GET .../progress` and the leaderboard always
  reflect the current moment, not the last snapshot.
- **Export delivery in development:** with no `SMTP_HOST` configured,
  `EMAIL` delivery still returns `201`/`200` with the file generated, but
  `emailedAt` stays `null` (the send is logged, not actually delivered).
  Don't treat a null `emailedAt` as a failure in dev.
- **Every export is requester-scoped.** `GET .../exports/:id` (and
  `/download`) 404s for anyone but the user who created it, even an ADMIN
  — export a fresh one instead of trying to share a job id across users.
