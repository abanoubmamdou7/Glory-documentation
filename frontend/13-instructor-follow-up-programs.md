# 13 — Follow-Up Programs (Instructor Dashboard, isolated)

> Audience: **the Instructor Dashboard frontend specifically** — a
> self-contained reference for the "Follow-up Programs" nav item, so this
> team doesn't need to also read the general dashboard's
> [05-follow-up-programs.md](05-follow-up-programs.md). Every endpoint
> below is identical to that doc's — this is the same API, just framed for
> one caller: **you are always an authenticated `INSTRUCTOR`**. Bearer
> token required on every endpoint; **reads need only a valid token**;
> writes (create + settings + rotation) need the `followup.manage`
> permission (ADMIN bypasses; a plain `INSTRUCTOR` account needs it
> granted explicitly — see "Getting access" below).

A follow-up program tracks one member's check-in (a workout log, an
In-Body test, a body-measurement update, a "student" record) against an
assigned instructor — `status` is `ON_TIME` / `MISSING` / `PENDING`.

## Getting access

A freshly created `INSTRUCTOR` employee has **no** permissions by default.
Reads work immediately (any valid token). To create a follow-up or touch
settings/rotation, grant `followup.manage` once:

```
PUT /instructors/:instructorId/permissions
{ "permissionKeys": ["followup.manage"] }
```

(Called by an ADMIN/STAFF account with `instructors.manage`, or log in as
the seeded super admin — ADMIN bypasses every permission check.) Seeded
test instructors: `coach1@glorytest.local` / `coach2@glorytest.local`,
password `Test1234`.

---

## The screens, top to bottom

| # | Screen | Call |
| --- | --- | --- |
| 1 | "Follow-Up Programs" list | `GET /follow-up-programs` |
| 2 | The 3 stat cards | `GET /follow-up-programs/stats` |
| 3 | "Export" button | `GET /follow-up-programs/export` |
| 4 | "+ Add Follow-Up" (Member + Follow-Up Type, then type-specific fields) | `POST /follow-up-programs` |
| 5 | Row "eye" view action | `GET /follow-up-programs/:id` |
| 6 | Follow-Up Program Setting toggles | `GET`/`PATCH /follow-up-programs/settings` |
| 7 | "Instructors Set in Active Use" + "Follow Another Instructors" | `GET`/`POST /follow-up-programs/settings/instructors` |
| 8 | Rotation row "eye" / Status switch | `GET`/`PATCH /follow-up-programs/settings/instructors/:id` |

---

## 1–3. The list, stats, export

```
GET /follow-up-programs?page=1&limit=10&search=&status=&expectedDate=&sortBy=&sortOrder=desc
```

- `search` matches **either** the member's or the assigned instructor's
  id/fullName/phone/email (both are checked, OR'd together) — build the
  search box against "ID, Name, Phone Number or E-mail" without worrying
  about which side of the follow-up it belongs to.
- `status` — `ON_TIME` / `MISSING` / `PENDING`.
- `expectedDate` — a single date (`"2026-03-30"`); matches that whole
  calendar day.
- `sortBy` — `code`, `expectedDate`, `status`, or `createdAt`.

```json
{ "id": "...", "code": 3564,
  "recordType": "WORKOUT",
  "status": "ON_TIME",
  "expectedDate": "2026-03-30T00:00:00.000Z",
  "actualEntryDate": null,
  "member": { "id": "...", "fullName": "Dwight Rohan", "email": "...", "phone": "...", "avatarUrl": null },
  "assignedInstructor": { "id": "...", "fullName": "Coach Ahmed", "email": "...", "phone": "...", "avatarUrl": null },
  "package": null,
  "createdAt": "..." }
```

`code` is the "ID Follow-Up" column. `GET /follow-up-programs/stats` →
`{ total, missing, onTime }`, each `{ count, changePct }` (% change vs 7
days ago). `GET /follow-up-programs/export` is the same query params minus
pagination, returned as `text/csv` (UTF-8 BOM, safe for Arabic names in
Excel).

## 4. "+ Add Follow-Up"

Two fields are always sent:

```json
{ "memberId": "...", "recordType": "IN_BODY_TEST" }
```

- **Member** — populate the dropdown from
  `GET /instructor-dashboard/members?search=` (the same gym-wide member
  list powering your Dashboard home's "Total Members" widget).
- **Follow-Up Type** — one of `WORKOUT`, `BODY_MEASUREMENTS`,
  `IN_BODY_TEST`, `STUDENT`.

When Follow-Up Type is **"In-Body Test"**, the form expands to 8 more
inputs — send them alongside the two base fields, all optional:

```json
{
  "memberId": "...",
  "recordType": "IN_BODY_TEST",
  "weight": "80.50",
  "muscleMass": "35.20",
  "bodyFat": "20.10",
  "bodyWater": "55.00",
  "visceralFat": "7.00",
  "bmi": "24.50",
  "bmr": "1750.00",
  "metabolicAge": 25
}
```

`weight`/`muscleMass`/`bodyFat`/`bodyWater`/`visceralFat`/`bmi`/`bmr` are
decimal strings (2 dp); `metabolicAge` is a plain integer. **"Body
Measurements"** is expected to want the same 8 fields (same underlying
record, different type) — send them the same way if/when that state is
built. **"Workout"** and **"Student"** currently take just the two base
fields — no extra inputs exist for them yet.

You don't send `instructorId` at all in the normal instructor-dashboard
flow — it **defaults to you**, the logged-in instructor. `expectedDate`
and `packageId` are optional and not present on this form; leave them out.

**201 response:** the created program, plus `bodyRecord` — `null` for
`WORKOUT`/`STUDENT`, or the saved metrics for `IN_BODY_TEST`/
`BODY_MEASUREMENTS`:

```json
{ "id": "...", "code": 3565, "recordType": "IN_BODY_TEST",
  "status": "ON_TIME", "actualEntryDate": "2026-08-24T...",
  "member": { "...": "..." }, "assignedInstructor": { "id": "you", "fullName": "Coach Ahmed" },
  "bodyRecord": { "id": "...", "type": "IN_BODY_TEST", "weight": "80.5", "muscleMass": "35.2", "...": "..." } }
```

`status` is always `ON_TIME` and `actualEntryDate` is always "now" on
create — this form logs a follow-up happening **right now**, it isn't a
scheduler. **Errors:** `400` unknown `memberId`; `403` missing
`followup.manage`.

## 5. View a follow-up

`GET /follow-up-programs/:id` — same shape as the create response
(includes `bodyRecord` when relevant), plus `aiEvaluation`/`aiFeedback`/
`delaySentEmails` for a fuller detail panel than the list row shows.
There's no edit or delete for an existing program.

## 6. Follow-Up Program Setting

```
GET /follow-up-programs/settings
```

Never 404s — auto-creates a default (all `false`) row on first read:

```json
{ "id": "...", "branchId": null,
  "notifyOnMissedTasks": false,
  "followUpCycleTracking": false,
  "autoInstructorRotation": false }
```

`PATCH /follow-up-programs/settings` — send only the toggle(s) that
changed, e.g. `{ "autoInstructorRotation": true }`.

## 7–8. Instructor rotation

```
GET /follow-up-programs/settings/instructors?page=1&limit=10
```

Paginated, ordered by rotation position:

```json
{ "id": "...", "instructorId": "...", "joinedDate": "...",
  "active": true, "order": 0,
  "instructor": { "id": "...", "fullName": "Coach Ahmed", "email": "...", "avatarUrl": null } }
```

This table's own "ID" column is just the row's 1-based position in the
array — not a stored field. `POST /follow-up-programs/settings/instructors`
— "Follow Another Instructors":

```json
{ "instructorIds": ["usr_123", "usr_456"] }
```

Adds one or more instructors to the end of the rotation (`active: true` by
default). **201** returns only the newly-added rows — ids already in the
rotation are silently skipped, not an error, unless *every* id given is
already there (then `400`). `GET .../instructors/:id` reads one row;
`PATCH .../instructors/:id` with `{ "active": false }` is the "Status"
switch — there's no delete for a rotation row.

---

## Gotchas checklist

1. A fresh `INSTRUCTOR` login has zero permissions — reads work, writes
   403 until `followup.manage` is granted (see "Getting access" above).
2. `instructorId` is **not** in the "Add Follow-Up" payload you build —
   it's inferred server-side from your own token.
3. Only `IN_BODY_TEST` has confirmed extra fields; `BODY_MEASUREMENTS` is
   a reasonable assumption, `WORKOUT`/`STUDENT` have none yet.
4. "ID Follow-Up" means two different things depending on the table: the
   main list's `code` (a real column) vs. the rotation table's 1-based row
   position (computed from array order) — don't confuse the two.
5. There's no edit/delete for an existing follow-up program.
6. `expectedDate` filtering matches a whole calendar day, not an exact
   timestamp.
