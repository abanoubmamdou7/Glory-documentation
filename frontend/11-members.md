# 11 — Members (Instructor Dashboard)

> Audience: **web dashboard frontend**. Covers the Instructor Dashboard's
> own **"Members"** nav item: the "Members Information" list + stat cards,
> and a member's profile page's **In-Body Test / Body Measurements** panel
> (under the "Follow-Up History" tab). Reads need `members.read`; body
> record writes need `members.manage` (ADMIN bypasses).

⚠ **Scope note, read this first:** this is a **Phase 4 (Members & Family)
slice pulled forward**, same as Instructor Dashboard/Workouts/Coach Chat
before it — built from exactly the 3 screens provided (list, profile
shell + In-Body Test tab, Add In-Body Test modal), not the full Members &
Family phase. Specifically **not built** here, because their screens
weren't provided:
- **"Add New Members"** (creating a member) — no screen shown.
- **"Edit Profile"** (the button exists on the profile header, but its
  form content wasn't shown).
- The list's row-level **edit**/**delete** icons — same reason.
- The profile's **"Profile Info"**, **"Bookings"**, **"Workouts"** tabs'
  own content — only **"Follow-Up History" → "In-Body Test"** was
  expanded in the source screens. `GET /members/:id` returns the full
  member record (a reasonable "Profile Info" tab backing, even though the
  exact layout wasn't shown); **Bookings** has no dashboard-side
  by-member listing yet (only the member's own `GET /mobile/bookings`
  exists); **Workouts** should reuse the already-built
  `GET /workout-assignments?memberId=` ([06-workouts.md](06-workouts.md)).

This is **distinct** from the Dashboard-home widget documented in
[09-instructor-dashboard.md](09-instructor-dashboard.md) — that one
(`GET /instructor-dashboard/members`) is a smaller "remaining days"
preview table; this "Members" nav item is the fuller list with its own
stat cards, Status column/filter, and drill-into-profile flow.

| # | Method & path | Screen element |
| --- | --- | --- |
| 1 | `GET /members/stats` | The 4 stat cards |
| 2 | `GET /members` | The "Members Information" list |
| 3 | `GET /members/export` | The list's "Export" button |
| 4 | `GET /members/:id` | Profile header (name/avatar/status/join date) |
| 5 | `POST /members/:memberId/body-records` | "+ Add New In-Body Test Record" |
| 6 | `GET /members/:memberId/body-records` | The In-Body Test Record list |
| 7 | `GET /members/:memberId/body-records/:id` | One record |
| 8 | `PATCH /members/:memberId/body-records/:id` | The record's edit (pencil) icon |
| 9 | `DELETE /members/:memberId/body-records/:id` | The record's delete (trash) icon |

---

## 1. `GET /members/stats` — the 4 stat cards

```json
{ "total": { "count": 45627, "changePct": 10 },
  "active": { "count": 40, "changePct": 10 },
  "inactive": { "count": 70, "changePct": -10 },
  "freezed": { "count": 13, "changePct": -10 } }
```

Same `{ count, changePct }` shape and "vs 7 days ago" approximation as
Team Management's stats (member status changes aren't historized either,
so this is a `createdAt`-based baseline, not a true historical diff).

## 2. `GET /members` — the list

Query: `page`, `limit`, `search` ("Search by Members (ID, Name, Phone
Number or E-mail)" — matches id/memberCode/fullName/email/phone), `status`
(`ACTIVE`|`INACTIVE`|`FREEZED` — the Status filter dropdown; also drives
the row status badge), `sortBy` (`memberCode`|`fullName`|`status`|
`createdAt`), `sortOrder`.

```json
{ "id": "...", "memberCode": "45627", "fullName": "Dwight Rohan",
  "email": "Reyna_King51@hotmail.com", "avatarUrl": null,
  "phone": "501234567", "phoneCountryCode": "+966",
  "status": "ACTIVE", "createdAt": "..." }
```

Column mapping: "ID Number" = `memberCode`; "Members Details" = avatar +
`fullName` + `email`; "Mobile Number" = flag (derive from
`phoneCountryCode`) + `phoneCountryCode` + `phone`; "Status" = badge,
color per value; "Action" eye icon → `GET /members/:id`. The edit/delete
icons have no backing endpoint yet (see the scope note above).

## 3. `GET /members/export` — "Export"

CSV, honors `search`/`status`. Columns: ID Number, Full Name, Email,
Mobile Number, Status, Joined At. UTF-8 BOM for Excel Arabic support.

## 4. `GET /members/:id` — profile header

Returns the full `Member` record (every scalar field except the password
hash — never returned). For the header shown in the screens: `fullName`,
`avatarUrl`, `status`, `createdAt` ("Oct 10, 2025" — the join date shown
under the name). Everything else on the record (nationality, city,
gender, dateOfBirth, maritalStatus, …) is available for a "Profile Info"
tab whenever its real layout is provided.

## 5. `POST /members/:memberId/body-records` — "Add New In-Body Test Record"

Same endpoint for **both** In-Body Test and Body Measurements — they're
one model (`BodyRecord`), differentiated by `type`.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `type` | `IN_BODY_TEST`\|`BODY_MEASUREMENT` | no | Default `IN_BODY_TEST` |
| `weight` | decimal string, 2dp | no | |
| `muscleMass` | decimal string, 2dp | no | |
| `bodyFat` | decimal string, 2dp | no | |
| `bodyWater` | decimal string, 2dp | no | |
| `visceralFat` | decimal string, 2dp | no | |
| `bmi` | decimal string, 2dp | no | |
| `bmr` | decimal string, 2dp | no | |
| `metabolicAge` | int | no | |
| `pdfUrl` | string | no | From `POST /uploads` — the "+ Pdf Upload" alternative flow |

Every metric is optional — the modal's "Choose … presets … or upload your
own" copy suggests a record can be PDF-only, with no manually entered
values.

**201:**

```json
{ "id": "...", "type": "IN_BODY_TEST",
  "weight": "80.5", "muscleMass": "35.2", "bodyFat": "20.1",
  "bodyWater": "55", "visceralFat": "7", "bmi": "24.5", "bmr": "1750",
  "metabolicAge": 25, "pdfUrl": null,
  "createdBy": { "id": "...", "fullName": "ahmad UI-UX" },
  "recordedAt": "..." }
```

> ⚠ **Documented inference:** the card shows **"Created By"** (a name) and
> **"Registered By"** (a *date* — "Apr 25, 2026"), which reads like a
> label mismatch (`BodyRecord` has no separate "registered by" actor,
> only `createdBy`/`recordedAt`). Mapped "Created By" → `createdBy.fullName`,
> "Registered By" → `recordedAt` (i.e. treat it as "Registered **On**").
> Flag back if the real intent differs once confirmed.

## 6. `GET /members/:memberId/body-records` — the record list

Query: `page`, `limit`, `type` (the left sub-nav: **Body Measurements** vs
**In-Body Test**). Ordered newest-first (`recordedAt` desc).

## 7. `GET /members/:memberId/body-records/:id`

`404` if the record doesn't belong to `memberId` in the URL (ownership is
scoped by the path, not just the record id).

## 8. `PATCH /members/:memberId/body-records/:id` — edit (pencil icon)

True partial — send only the fields being changed.

## 9. `DELETE /members/:memberId/body-records/:id` — delete (trash icon)

No FK dependents — a plain delete, no 409 case.

---

## Gotchas checklist

1. This "Members" nav item is **not** the same as the Dashboard-home
   members widget ([09-instructor-dashboard.md](09-instructor-dashboard.md))
   — don't conflate the two `GET .../members`-shaped endpoints.
2. Member create/edit and the list's edit/delete row actions have no API
   yet — see the scope note at the top before wiring those buttons.
3. `POST`/`PATCH .../body-records` accept **decimal strings**, never JS
   numbers, same convention as every money/weight field in this API.
4. A body record's ownership is checked against the `:memberId` in the
   URL path — fetching a real record id under the wrong member's path
   404s, it doesn't leak data.
5. `type` on a body record can't be changed after creation implicitly —
   `PATCH` accepts it like any other field, but there's no dedicated
   "convert between In-Body Test and Body Measurement" flow; treat it as
   a normal editable field if you need to.
