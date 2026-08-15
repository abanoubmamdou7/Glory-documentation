# 12 — Workouts (Instructor Dashboard, isolated)

> Audience: **the Instructor Dashboard frontend specifically** — a
> self-contained reference for the "Workouts" nav item, so this team
> doesn't need to also read the general dashboard's
> [06-workouts.md](06-workouts.md). Every endpoint below is identical to
> that doc's — this is the same API, just framed for one caller: **you are
> always an authenticated `INSTRUCTOR`**. Bearer token required on every
> endpoint; **reads need only a valid token**; writes
> (create/update/delete/replace-stages) need the `workouts.manage`
> permission (ADMIN bypasses; a plain `INSTRUCTOR` account needs it
> granted explicitly — see "Getting access" below).

⚠ **One fact worth knowing up front:** `Workout` is a **shared gym-wide
catalog**, not private to whichever instructor created it — like Videos
Library. Any instructor (or staff/admin) with `workouts.manage` can see
and edit any workout, not just their own. The list you'll build isn't
"my workouts", it's "the gym's workouts" (which will usually mean mostly
ones this instructor authored, but not exclusively).

## Getting access

A freshly created `INSTRUCTOR` employee has **no** permissions by default.
Reads work immediately (any valid token). To create/edit workouts, grant
`workouts.manage` once:

```
PUT /instructors/:instructorId/permissions
{ "permissionKeys": ["workouts.manage"] }
```

(Called by an ADMIN/STAFF account with `instructors.manage`, or log in as
the seeded super admin — ADMIN bypasses every permission check, so it's
the fastest way to test end-to-end.) Seeded test instructors:
`coach1@glorytest.local` / `coach2@glorytest.local`, password `Test1234`.

---

## The screens, top to bottom

| # | Screen | Call |
| --- | --- | --- |
| 1 | "Workout Information" list | `GET /workouts` |
| 2 | List filters (Type / Duration / Level / Row Per Page) | query params on the same call |
| 3 | "Export" button | `GET /workouts/export` |
| 4 | "+ Add New Workout" → Step 1 "Workout Identity" | held client-side |
| 5 | Step 2 "Workout Instructions" | held client-side |
| 6 | Step 3 "Add Members" | held client-side |
| 7 | Final submit ("Add Workouts") | **one** `POST /workouts` call carrying steps 4–6 together |
| 8 | Row "view" (eye icon) | `GET /workouts/:id` |
| 9 | Row "edit" (pencil icon) | `PATCH /workouts/:id` (+ `PUT /workouts/:id/instructions` if stages changed) |
| 10 | Row "delete" (trash icon) | `DELETE /workouts/:id` |
| 11 | Adding one more member to an already-created workout | `POST /workout-assignments` |

---

## 1–3. The list screen

```
GET /workouts?page=1&limit=10&search=&type=&level=&minDurationDays=&maxDurationDays=&sortBy=&sortOrder=desc
```

- `search` — matches the "ID Number" column (numeric `code`, e.g. `3565`),
  the underlying `id`, or the name (EN/AR).
- `type` — one of `STRENGTH_TRAINING`, `WEIGHT_TRAINING`,
  `FLEXIBILITY_MOBILITY`, `REHABILITATION_LOW_IMPACT`,
  `FUNCTIONAL_TRAINING`, `CARDIO`.
- `level` — `BEGINNER`, `INTERMEDIATE`, `MASTER`.
- `minDurationDays` / `maxDurationDays` — the "Duration" filter. It's a
  plain inclusive range, not a fixed set of options — build whatever
  preset buckets the design calls for (e.g. "Under 15 Days") on top of
  this by translating the selected bucket into a min/max pair before
  calling the API.
- `sortBy` — `code`, `nameEn`, `type`, `level`, `durationDays`, or
  `createdAt`. `sortOrder` — `asc`/`desc`.

Each row:

```json
{ "id": "...", "code": 3565, "nameEn": "Cardio Program", "nameAr": "...",
  "type": "CARDIO", "level": "BEGINNER", "durationDays": 40,
  "createdBy": { "id": "...", "fullName": "Coach Ahmed" },
  "_count": { "assignments": 2, "instructions": 3 } }
```

`code` is the "ID Number" column. `_count.assignments` = how many members
this workout is currently assigned to; `_count.instructions` = how many
stages it has — useful for a quick summary without a second call.

"Export" is `GET /workouts/export` with the **same** query params as the
list minus pagination/sort — it always returns every matching row as CSV
(`text/csv`, UTF-8 BOM so Excel shows Arabic names correctly).

## 4–7. "+ Add New Workout" — one call at the end

The 3-step stepper is **pure client-side state**. Nothing is created
until the very last "Add Workouts" click on step 3 — build the full
payload across all three steps in memory and submit once:

```json
POST /workouts
{
  "nameEn": "Cardio Program",
  "nameAr": "برنامج كارديو",
  "type": "CARDIO",
  "level": "BEGINNER",
  "durationDays": 40,

  "instructions": [
    { "stepNumber": 1, "instructionEn": "Leg Extension", "instructionAr": "تمديد الساق",
      "videoIds": ["<id from POST /video-library>"] },
    { "stepNumber": 2, "instructionEn": "Cardio", "instructionAr": "كارديو" }
  ],

  "memberIds": ["<member id>", "<member id>"]
}
```

- **Step 1 fields:** `nameEn` (required), `nameAr`, `type` (required),
  `level` (required), `durationDays`.
- **Step 2, `instructions[]`** (optional): one object per stage.
  `stepNumber` must be unique in the array (1-based). `videoIds` is
  optional per stage and can hold more than one video id — every id must
  already exist in the Video Library or the **whole request fails**
  (nothing partially created). The wizard's "Number Of Videos" column is
  just `videoIds.length` — no API call needed to compute it while staging.
- **Step 3, `memberIds[]`** (optional): every member selected in "Assign
  Members". For each one, a `WorkoutAssignment` is created automatically —
  **you don't call anything else for this step.** `instructorId` on each
  assignment defaults to **you** (the logged-in instructor);
  `durationDays` inherits the workout's own value; `status` starts
  `UPCOMING`. An unknown member id fails the whole request with `400`
  before anything is created.
- The "Select Member Name" dropdown itself has no endpoint of its own —
  use `GET /instructor-dashboard/members?search=` (the same gym-wide
  member list that powers your Dashboard home's "Total Members" widget).
  "+ Add New Member" (creating a brand-new member inline) has no backing
  endpoint yet.

**201 response** — the created workout, stages included with each stage's
videos flattened (not the raw junction shape), plus `code`:

```json
{ "id": "...", "code": 3566, "nameEn": "Cardio Program", "nameAr": "...",
  "type": "CARDIO", "level": "BEGINNER", "durationDays": 40,
  "createdBy": { "id": "...", "fullName": "Coach Ahmed" },
  "_count": { "assignments": 2 },
  "instructions": [
    { "id": "...", "stepNumber": 1, "instructionEn": "Leg Extension", "instructionAr": "تمديد الساق",
      "videos": [ { "id": "...", "titleEn": "...", "videoUrl": "...", "thumbnailUrl": "...", "duration": "02:10" } ] },
    { "id": "...", "stepNumber": 2, "instructionEn": "Cardio", "instructionAr": "كارديو", "videos": [] }
  ] }
```

## 8. View a workout

`GET /workouts/:id` — same shape as the create response.

## 9. Edit a workout

Two separate calls, on purpose:
- `PATCH /workouts/:id` — scalars only (`nameEn`/`nameAr`/`type`/`level`/
  `durationDays`).
- `PUT /workouts/:id/instructions` — **replaces the whole stage list.**
  To edit, add, remove, or reorder even one stage, resend the full
  desired `instructions[]` array (same shape/validation as create).

Member assignments are edited separately too — see §11 below; there's no
single "update everything" call for an existing workout the way create
covers all three steps at once.

## 10. Delete a workout

`DELETE /workouts/:id` → **`409`** if any member is still assigned it.
Remove/reassign those first (`DELETE /workout-assignments/:id`).

## 11. Add one more member later

`POST /workout-assignments` does exactly what `memberIds[]` does on
create, one member at a time — use it for adding someone to a workout
that already exists (outside the create wizard):

```json
{ "workoutId": "...", "memberId": "...", "instructorId": "..." }
```

`instructorId` is optional here too but **not auto-defaulted** on this
endpoint the way `memberIds[]` on create is — pass your own id explicitly
if you want yourself as the assigned coach. `status`/`startDate`/
`endDate`/`durationDays`/`suggestedWeight` are all optional too.

To remove someone: `DELETE /workout-assignments/:id` (no confirmation/FK
blocker — it's a plain delete).

---

## Gotchas checklist

1. The wizard is **one** `POST /workouts` call at the end — not three
   separate calls per step.
2. `code` is auto-generated — display it, never send it on create.
3. A video has to be attached to a stage (`videoIds`) **and** the workout
   has to be assigned to a member before anything shows up in that
   member's mobile "تماريني" — uploading a video alone does nothing
   visible.
4. `PATCH /workouts/:id` never touches stages or members — always
   `PUT /workouts/:id/instructions` for stages, `POST`/
   `DELETE /workout-assignments` for members.
5. This is a **shared catalog** — don't build the list as if it's
   filtered to "workouts I created"; it isn't, by design.
6. A fresh `INSTRUCTOR` login has zero permissions — reads work, writes
   403 until `workouts.manage` is granted (see "Getting access" above).
7. `workoutId`/`memberId` on an existing assignment can't be changed —
   delete and recreate to move a member to a different program.
