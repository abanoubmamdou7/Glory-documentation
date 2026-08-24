# 05 — Follow-Up Programs

> Audience: **web dashboard frontend** — also the exact API behind the
> **Instructor Dashboard's** "Follow-up Programs" nav item. Covers the main
> list + stats screen, "Add Follow-Up", and **Follow-Up Program Setting**
> (toggles + instructor rotation). Reads need only a valid token; writes need
> `followup.manage` (ADMIN bypasses).

A follow-up program tracks one member's check-in (a workout log, an In-Body
test, a body-measurement update, a "student" record) against an assigned
instructor — `status` is `ON_TIME` / `MISSING` / `PENDING`. Individual
programs can now be created via **"Add Follow-Up"** (§4 below); there is
still no edit/delete for an existing one — the only per-row action besides
create is the "eye" view icon.

| # | Method & path | Screen |
| --- | --- | --- |
| 1 | `GET /follow-up-programs` | The main list + search/filter bar |
| 2 | `GET /follow-up-programs/stats` | The 3 stat cards |
| 3 | `GET /follow-up-programs/export` | "Export" button |
| 4 | `POST /follow-up-programs` | "Add Follow-Up" |
| 5 | `GET /follow-up-programs/:id` | Row's "eye" view action |
| 6 | `GET /follow-up-programs/settings` | Follow-Up Program Setting toggles |
| 7 | `PATCH /follow-up-programs/settings` | Toggling any of the three switches |
| 8 | `GET /follow-up-programs/settings/instructors` | "Instructors Set in Active Use" table |
| 9 | `POST /follow-up-programs/settings/instructors` | "Follow Another Instructors" modal |
| 10 | `GET /follow-up-programs/settings/instructors/:id` | Row's "eye" view action |
| 11 | `PATCH /follow-up-programs/settings/instructors/:id` | The rotation table's "Status" switch |

---

## 1. `GET /follow-up-programs` — the main list

Query: `page`, `limit`, `search` (matches **either** the member's **or** the
assigned instructor's id/name/phone/email — per the newer screens' "Search
By Members ( ID , Name , Phone Number Or E-Mail )"; the older "search by
instructor" behavior still works too, both are OR'd together), `status`
(`ON_TIME`|`MISSING`|`PENDING`), `expectedDate` (single date — matches
records whose `expectedDate` falls on that calendar day), `sortBy`
(`code`|`expectedDate`|`status`|`createdAt`), `sortOrder`.

```json
{ "id": "...", "code": 3564,
  "recordType": "WORKOUT",
  "status": "ON_TIME",
  "expectedDate": "2026-03-30T00:00:00.000Z",
  "actualEntryDate": null,
  "aiEvaluation": null, "aiFeedback": null, "delaySentEmails": 0,
  "member": { "id": "...", "fullName": "Reyna King", "email": "Reyna_King51@hotmail.com", "phone": "...", "avatarUrl": null },
  "assignedInstructor": { "id": "...", "fullName": "Dwight Rohan", "email": "Reyna_King51@hotmail.com", "phone": "...", "avatarUrl": null },
  "package": null,
  "createdAt": "...", "updatedAt": "..." }
```

Card/row mapping: **"ID Follow-Up"** column = `code` (a short auto-incrementing
number, not the internal `id`); **"Assigned Instructors"** = `assignedInstructor`
(avatar + name + email — `null` if unassigned); **"Member Details"** =
`member`; **"Expected Date"** = `expectedDate`; **"Status"** = `status` badge
(green `ON_TIME`, red `MISSING`, yellow `PENDING`).

> The design's per-row checkboxes have no associated bulk-action button in
> the current screens (only the individual "eye" view icon and the list-wide
> "Export") — no bulk endpoint exists yet. Ask if the product adds one.

## 2. `GET /follow-up-programs/stats` — the 3 cards

```json
{ "success": true, "data": {
  "total":   { "count": 123, "changePct": 4.3 },
  "missing": { "count": 40,  "changePct": 10.0 },
  "onTime":  { "count": 70,  "changePct": -10.0 }
} }
```

`changePct` = % change vs 7 days ago (approximated from `createdAt` — status
history isn't tracked, same caveat as the Team Management stats cards).

## 3. `GET /follow-up-programs/export` — "Export"

Honors the list's `search`/`status`/`expectedDate` filters; **no pagination**.
Returns `text/csv` with `Content-Disposition: attachment` and a **UTF-8 BOM**
(Arabic-safe in Excel). Handle it as a blob download, same as the Team
Management exports:

```ts
const res = await fetch(url, { headers: { Authorization: `Bearer ${t}` } });
const blob = await res.blob();
```

Columns: ID Follow-Up, Assigned Instructor, Instructor Email, Member Name,
Member Email, Expected Date, Status.

## 4. `POST /follow-up-programs` — "Add Follow-Up"

**Permission:** `followup.manage`.

The form's two base fields, always sent:

| Field | Maps to | Notes |
| --- | --- | --- |
| Member (dropdown) | `memberId` (required) | Populate from `GET /instructor-dashboard/members?search=` |
| Follow-Up Type (dropdown) | `recordType` (required) | `WORKOUT` \| `BODY_MEASUREMENTS` \| `IN_BODY_TEST` \| `STUDENT` |

When **Follow-Up Type = "In-Body Test"** (confirmed by the provided
screens), the form additionally shows 8 metric inputs — send them as-is,
all optional decimal strings (2 dp) except `metabolicAge` (a plain int):
`weight`, `muscleMass`, `bodyFat`, `bodyWater`, `visceralFat`, `bmi`,
`bmr`, `metabolicAge`.

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

⚠ **Documented inference — the other 3 Follow-Up Types weren't shown
expanded in the source screens:**
- **"Body Measurements"** almost certainly wants the *same* metric fields
  (it's the same underlying record, just a different type) — the API
  already accepts them for this type too, so wire it that way unless a
  real screen says otherwise.
- **"Workout"** and **"Student"** currently accept only the two base
  fields (`memberId`/`recordType`) — no extra fields are collected or
  stored for them yet. If their own "Add Follow-Up" states turn out to
  need extra inputs, this endpoint will need extending — check back here
  before assuming it's already covered.

**What happens server-side:** for `IN_BODY_TEST`/`BODY_MEASUREMENTS`, the
metric fields are saved as a genuine `BodyRecord` (the same record type
`GET /members/:id/body-records` reads — see
[11-members.md](11-members.md)), linked back to this follow-up. This is
recorded as happening **right now**, not scheduled for later — there's no
date/status picker on the form, so the server sets `actualEntryDate` to
today and `status` to `ON_TIME` automatically. `instructorId` defaults to
the caller when they're themselves an `INSTRUCTOR` (the normal case on the
Instructor Dashboard); `STAFF`/`ADMIN` can pass it explicitly.

**201 response:** the created program, same shape as a list item plus
`bodyRecord` (`null` for `WORKOUT`/`STUDENT`, or the linked record's full
field set for `IN_BODY_TEST`/`BODY_MEASUREMENTS`).

**Errors:** `400` unknown `memberId`/`packageId`, or an `instructorId` that
isn't a real `INSTRUCTOR`; `403` missing `followup.manage`.

## 5. `GET /follow-up-programs/:id`

Same shape as a list item, plus `bodyRecord` (see §4) — this is the "eye"
icon's detail view, and also includes `aiEvaluation`/`aiFeedback`/
`delaySentEmails` for a more detailed panel than the table row shows.

---

## Follow-Up Program Setting

### 6–7. The three toggles

`GET /follow-up-programs/settings` — auto-creates a default (all `false`) row
on first read, so it never 404s:

```json
{ "id": "...", "branchId": null,
  "notifyOnMissedTasks": false,
  "followUpCycleTracking": false,
  "autoInstructorRotation": false }
```

`PATCH /follow-up-programs/settings` — send only the toggle(s) that changed:

```json
{ "autoInstructorRotation": true }
```

| Field | Screen toggle |
| --- | --- |
| `notifyOnMissedTasks` | "Notify on missed tasks" |
| `followUpCycleTracking` | "Follow-up cycle tracking" |
| `autoInstructorRotation` | "Automatic instructors rotation" |

> `branchId` is always `null` in the current screens (there's no branch
> selector on this settings page) — it's a global default. Per-branch
> overrides aren't exposed by this API yet.

### 8. `GET /follow-up-programs/settings/instructors` — "Instructors Set in Active Use"

Paginated, ordered by rotation position (fair-rotation order, ascending):

```json
{ "id": "...", "instructorId": "...", "joinedDate": "2026-03-30T...",
  "active": true, "order": 0,
  "instructor": { "id": "...", "fullName": "Dwight Rohan", "email": "...", "avatarUrl": null } }
```

The table's **"ID Follow-Up"** column for this list is simply the row's
**1-based position** in the returned (already-ordered) array — there's no
separate code field for rotation rows. **"Instructors Details"** =
`instructor`; **"Joined Date"** = `joinedDate`; **"Status"** toggle = `active`.

### 9. `POST /follow-up-programs/settings/instructors` — "Follow Another Instructors"

```json
{ "instructorIds": ["usr_123", "usr_456"] }
```

Accepts one or more ids from the "Select Instructors" dropdown (populate that
dropdown from your existing Team Management instructor list —
`GET /instructors`, see [02-team-management.md](02-team-management.md)).
New rows join at the end of the rotation order (`active: true` by default).

**201:** the newly-created rotation rows (hydrated, same shape as the list).

**400 cases:** any id that doesn't exist; any id belonging to a non-INSTRUCTOR
user; every selected id is already in the rotation (partial overlap is
allowed — only the new ones get added, unless *all* are duplicates).

### 10. `GET /follow-up-programs/settings/instructors/:id`

One rotation row, same shape as a list item.

### 11. `PATCH /follow-up-programs/settings/instructors/:id` — the "Status" switch

```json
{ "active": false }
```

Toggling `active` off removes the instructor from automatic rotation
assignment without deleting their history — there is no delete/remove
endpoint (the design shows no trash icon on this table, only the toggle and
the "eye" view).

---

## Gotchas checklist

1. There's still no **edit/delete** for an existing follow-up program —
   only create (§4) and read; the design shows no such action.
2. **"ID Follow-Up" means two different things** depending on the table:
   the main list's `code` field (a real DB column) vs. the rotation table's
   1-based row position (computed client-side from array order) — don't
   confuse the two.
3. `expectedDate` filter matches a **whole day**, not an exact timestamp —
   send just the date part (`"2026-03-30"`).
4. `GET /follow-up-programs/settings` never 404s — first call creates the
   default row automatically.
5. Adding instructors to rotation is **idempotent-safe but not silent**: if
   you select 3 and 1 is already in the rotation, the other 2 are added and
   the response only contains those 2 — reconcile your local state from the
   response, not from what you sent.
6. "Add Follow-Up" only collects extra metric fields for `IN_BODY_TEST` per
   the confirmed screens — `BODY_MEASUREMENTS` accepts the same fields as a
   reasonable inference, but `WORKOUT`/`STUDENT` currently create a bare
   program with no linked data. Double check against real screens for those
   two before assuming more is captured.
7. A created program's `status` is always `ON_TIME` and `actualEntryDate`
   always "now" — this endpoint records a follow-up that's happening live,
   it doesn't schedule a future one.
