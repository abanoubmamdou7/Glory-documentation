# 05 — Follow-Up Programs

> Audience: **web dashboard frontend**. Covers the **Follow-up Programs** nav
> item: the main list + stats screen, and **Follow-Up Program Setting**
> (toggles + instructor rotation). Reads need only a valid token; writes need
> `followup.manage` (ADMIN bypasses).

A follow-up program tracks one member's scheduled check-in (a workout log, an
In-Body test, a body-measurement update, …) against an assigned instructor
and an expected date — `status` is `ON_TIME` / `MISSING` / `PENDING`.
Programs themselves are system/AI-generated (per the design's "aiEvaluation"
/ "aiFeedback" fields on the record) — **this API does not create or edit
individual programs**, only reads them and manages the settings around them,
matching the screens (no "add program" or "edit program" button anywhere in
the design; the only per-row action is the "eye" view icon).

| # | Method & path | Screen |
| --- | --- | --- |
| 1 | `GET /follow-up-programs` | The main list + search/filter bar |
| 2 | `GET /follow-up-programs/stats` | The 3 stat cards |
| 3 | `GET /follow-up-programs/export` | "Export" button |
| 4 | `GET /follow-up-programs/:id` | Row's "eye" view action |
| 5 | `GET /follow-up-programs/settings` | Follow-Up Program Setting toggles |
| 6 | `PATCH /follow-up-programs/settings` | Toggling any of the three switches |
| 7 | `GET /follow-up-programs/settings/instructors` | "Instructors Set in Active Use" table |
| 8 | `POST /follow-up-programs/settings/instructors` | "Follow Another Instructors" modal |
| 9 | `GET /follow-up-programs/settings/instructors/:id` | Row's "eye" view action |
| 10 | `PATCH /follow-up-programs/settings/instructors/:id` | The rotation table's "Status" switch |

---

## 1. `GET /follow-up-programs` — the main list

Query: `page`, `limit`, `search` (matches the **assigned instructor's**
id/name/phone/email — per the design's "Search by instructors ( ID , Name ,
Phone Number Or E mail )"), `status` (`ON_TIME`|`MISSING`|`PENDING`),
`expectedDate` (single date — matches records whose `expectedDate` falls on
that calendar day), `sortBy` (`code`|`expectedDate`|`status`|`createdAt`),
`sortOrder`.

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

## 4. `GET /follow-up-programs/:id`

Same shape as a list item — this is the "eye" icon's detail view, and also
includes `aiEvaluation`/`aiFeedback`/`delaySentEmails` for a more detailed
panel than the table row shows.

---

## Follow-Up Program Setting

### 5–6. The three toggles

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

### 7. `GET /follow-up-programs/settings/instructors` — "Instructors Set in Active Use"

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

### 8. `POST /follow-up-programs/settings/instructors` — "Follow Another Instructors"

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

### 9. `GET /follow-up-programs/settings/instructors/:id`

One rotation row, same shape as a list item.

### 10. `PATCH /follow-up-programs/settings/instructors/:id` — the "Status" switch

```json
{ "active": false }
```

Toggling `active` off removes the instructor from automatic rotation
assignment without deleting their history — there is no delete/remove
endpoint (the design shows no trash icon on this table, only the toggle and
the "eye" view).

---

## Gotchas checklist

1. There's no create/edit for individual follow-up programs — they're system
   -generated; this API is read + settings/rotation management only.
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
