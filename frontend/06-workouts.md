# 06 — Workouts (dashboard)

> Audience: **web dashboard frontend**. Covers building **Workout program
> templates** (name, type, level, stages, videos) and **assigning** them to
> members — the piece that makes an exercise show up in the mobile app's
> "تماريني" ([mobile/06-workouts.md](../mobile/06-workouts.md)). Reads need
> only a valid token; writes need `workouts.manage` (ADMIN bypasses).

This is also the exact API behind the **Instructor Dashboard's "Workouts"
nav item** — the "Workout Information" list screen and the 3-step "Add New
Workout" wizard (**Workout Identity** → **Workout Instructions** → **Add
Members**), confirmed against real Figma screens. Same shared catalog
either way — like Videos Library, **not** instructor-scoped (any
authenticated user with `workouts.manage` manages any workout, unlike
Reservations which are personal to the calling instructor).

## The "Add New Workout" wizard is one API call, not three

The 3-step stepper is client-side state — nothing is persisted until the
final "Add Wotkout"/"Add Workouts" submit on step 3. That submit is a
**single** `POST /workouts` carrying everything collected across all three
steps: the identity fields (step 1), `instructions[]` (step 2), and
`memberIds[]` (step 3) — mirroring the same "single-call wizard" pattern
already used for [Team Management](02-team-management.md)'s employee
create. Don't call `POST /workouts` after step 1 and then patch it later —
build the full payload client-side across all three steps and submit once
at the end.

## How the pieces fit together

```
Video Library (already built)  →  a stage's "Video" box
        ↓ attach by id
Workout ("Cardio Program")  →  has stages (WorkoutInstruction, "المرحلة الاولة"...)
        ↓ assign to a member
WorkoutAssignment  →  THIS is what GET /mobile/workouts reads
```

Uploading a video, alone, does **not** make anything appear for a member —
it has to be attached to a stage of a Workout, and that Workout has to be
assigned to the member. See the mobile doc's "Gotchas" for the same
explanation from the member side.

| # | Method & path | Purpose |
| --- | --- | --- |
| 1 | `POST /workouts` | The whole wizard: create a program + stages + member assignments in one call |
| 2 | `GET /workouts` | "Workout Information" list — search/filter/sort |
| 3 | `GET /workouts/export` | The list's "Export" button — CSV |
| 4 | `GET /workouts/:id` | Get a program with its stages + videos |
| 5 | `PATCH /workouts/:id` | Update program scalars |
| 6 | `PUT /workouts/:id/instructions` | Replace the whole stage list |
| 7 | `DELETE /workouts/:id` | Delete a program |
| 8 | `POST /workout-assignments` | Add one more member to an **existing** program |
| 9 | `GET /workout-assignments` | List assignments (all members) |
| 10 | `GET /workout-assignments/:id` | Get one assignment |
| 11 | `PATCH /workout-assignments/:id` | Update status/dates/weight/instructor |
| 12 | `DELETE /workout-assignments/:id` | Remove an assignment |

---

## 1. `POST /workouts` — the whole wizard, one call

**Step 1, Workout Identity:**

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `nameEn` | string ≤150 | **yes** | |
| `nameAr` | string ≤150 | no | |
| `type` | `STRENGTH_TRAINING`\|`WEIGHT_TRAINING`\|`FLEXIBILITY_MOBILITY`\|`REHABILITATION_LOW_IMPACT`\|`FUNCTIONAL_TRAINING`\|`CARDIO` | **yes** | The list's "Workout Type" badge/filter |
| `level` | `BEGINNER`\|`INTERMEDIATE`\|`MASTER` | **yes** | The list's "Level" column/filter |
| `durationDays` | int ≥1 | no | e.g. `40` — the list's "Duration" column |

**Step 2, Workout Instructions — `instructions[]`** (optional, created
atomically, each one a "Step Two"/"Step Three"/... card):

```json
{ "stepNumber": 1, "instructionEn": "Leg Extension", "instructionAr": "تمديد الساق",
  "videoIds": ["<id(s) from POST /video-library>"] }
```

`stepNumber` must be unique within the array (1-based stage order, matches
the wizard's own running "Step Number" table). `videoIds` is optional per
stage and can list more than one video (the table's "Number Of
Videos" column is just `videoIds.length` — compute it client-side while
staging, no API round-trip needed until final submit); every id must
already exist in the Video Library or the whole request `400`s.

**Step 3, Add Members — `memberIds[]`** (optional): every member id
selected in the "Assign Members" table. For each one, a `WorkoutAssignment`
is created in the same transaction as the workout — `instructorId`
defaults to the caller if they're themselves an `INSTRUCTOR` (the normal
case here, since this screen lives on the instructor's own dashboard),
`durationDays` inherits the workout's own value, `status` defaults to
`UPCOMING`. Unknown member ids → `400` before anything is created (nothing
partially persists).

> The "Select Member Name" dropdown / member picker itself has no dedicated
> endpoint — reuse the already-built
> `GET /instructor-dashboard/members?search=` (gym-wide; matches
> id/memberCode/fullName/email/phone — same one powering that dashboard's
> "Total Members" widget). "+ Add New Member" (create a brand-new member
> inline) has no backing endpoint yet — Members & Family is Phase 4,
> not built.

**201:** the created program, stages included, each stage's videos
**flattened** (not the raw `{ video: {...} }` junction shape):

```json
{ "id": "...", "code": 3565, "nameEn": "Cardio Program", "nameAr": "...",
  "type": "CARDIO", "level": "BEGINNER", "durationDays": 40,
  "createdBy": { "id": "...", "fullName": "Super Admin" },
  "_count": { "assignments": 2 },
  "instructions": [
    { "id": "...", "stepNumber": 1, "instructionEn": "Cardio", "instructionAr": "كارديو",
      "videos": [ { "id": "...", "titleEn": "...", "videoUrl": "...", "thumbnailUrl": "...", "duration": "12:10" } ] },
    { "id": "...", "stepNumber": 2, "instructionEn": "Cardio", "instructionAr": "كارديو", "videos": [] }
  ] }
```

`code` is the list's short numeric "ID Number" column (e.g. `3565`,
auto-incrementing) — display it, never send it on create.

## 2. `GET /workouts` — the "Workout Information" list

Query: `page`, `limit`, `search` (id/**code**/nameEn/nameAr — "Search by
Workout (ID, Name)"), `type` ("Workout Type" filter), `level` ("Level"
filter), `minDurationDays`/`maxDurationDays` (inclusive range — backs the
"Duration" filter dropdown; ⚠ the design shows this as a dropdown without
specifying its preset buckets in the source screens, so this is a plain
min/max the frontend renders whatever ranges it wants on top of), `sortBy`
(`code`|`nameEn`|`type`|`level`|`durationDays`|`createdAt`), `sortOrder`.
List items include `_count.assignments`/`_count.instructions` (not the
full stage breakdown — call `GET /:id` for that).

> ⚠ The source screenshot's "Duration" column values mix days ("10 Days")
> and minutes ("40 Min") across rows — almost certainly a Figma
> placeholder/copy-paste artifact (same kind of thing as the Instructor
> Dashboard's "Remaining Assessment" block or Products' "Number of days"
> weight placeholder). Treated as `durationDays` throughout, matching the
> already-existing schema field and the row that *does* show "Days".

## 3. `GET /workouts/export` — the list's "Export" button

CSV, honors the same `search`/`type`/`level`/`minDurationDays`/
`maxDurationDays` filters as the list (no pagination — every matching
row). Columns: ID Number, Workout Name (EN), Workout Name (AR), Duration
(Days), Workout Type, Level, Created By, Assigned Members, Created At.
UTF-8 BOM prefix so Excel renders Arabic names correctly — same convention
as every other export in this API (Products, Team, Follow-Up Programs, …).

## 4. `GET /workouts/:id`

Same shape as the create response.

## 5. `PATCH /workouts/:id`

Scalars only (`nameEn`/`nameAr`/`type`/`level`/`durationDays`) — stages are
**not** editable here, use the dedicated endpoint below (same separation
Branches uses for `PATCH` vs `PUT .../images`); member assignments go
through `POST`/`DELETE /workout-assignments`, not here either.

## 6. `PUT /workouts/:id/instructions` — replace stages

```json
{ "instructions": [
  { "stepNumber": 1, "instructionEn": "Warmup", "videoIds": ["..."] },
  { "stepNumber": 2, "instructionEn": "Main Set", "videoIds": ["...", "..."] }
] }
```

**Replaces the entire stage list** — this is how you add, edit, remove, or
reorder stages: just resend the full desired array. Same validation as
create (unique `stepNumber`, all `videoIds` must exist).

## 7. `DELETE /workouts/:id`

**409** if any member is still assigned this program — remove/reassign those
first (`DELETE /workout-assignments/:id`).

---

## 8. `POST /workout-assignments` — add a member to an existing program

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `workoutId` | id | **yes** | |
| `memberId` | id | **yes** | |
| `instructorId` | id | no | Must be an existing `role=INSTRUCTOR` user |
| `status` | `UPCOMING`\|`IN_PROGRESS`\|`COMPLETED` | no | Default `UPCOMING` |
| `startDate` / `endDate` | date | no | |
| `durationDays` | int | no | Defaults to the workout's own `durationDays` |
| `suggestedWeight` | decimal string, 2dp | no | The initial target weight |

**201:** the assignment, hydrated with `workout`/`member`/`instructor`.
This is the exact row `GET /mobile/workouts` reads — set `status:
"IN_PROGRESS"` if you want the member to be able to log weight
immediately (mobile "اضافة وزن" 400s on `UPCOMING`/`COMPLETED`).

## 9–10. `GET /workout-assignments` / `GET /workout-assignments/:id`

List: filter `memberId`/`workoutId`/`status`. Both auth-only.

## 11. `PATCH /workout-assignments/:id`

`workoutId`/`memberId` are **not editable** — to reassign, delete and create
a new one instead (this mirrors treating an assignment as a specific
enrollment, not a mutable pointer). Everything else (`instructorId`/
`status`/`startDate`/`endDate`/`durationDays`/`suggestedWeight`) can change.

> **Setting a new `suggestedWeight` shifts the previous value into
> `suggestedWeightLast`** before saving — the exact mirror of what the
> mobile "اضافة وزن" action does to `userWeight`/`userWeightLast`. This is
> how staff periodically update the member's target weight across cycles.

## 12. `DELETE /workout-assignments/:id`

Removes the assignment — the exercise disappears from that member's
"تماريني". No FK dependents, so this is a plain delete (no 409 case).

---

## Gotchas checklist

1. The 3-step "Add New Workout" wizard is **one** `POST /workouts` call at
   the very end (step 1 + `instructions[]` + `memberIds[]` together) — the
   stepper is purely client-side state, nothing persists until "Add
   Workouts" is actually clicked.
2. Uploading a video alone does nothing visible to any member — it has to
   be attached to a stage (`videoIds` on create or on the replace-stages
   call) of a `Workout`, and that `Workout` has to be assigned to a member
   (via `memberIds[]` on create, or `POST /workout-assignments` afterward).
3. `code` (the "ID Number" column) is auto-generated — display it, never
   send it; `search` matches it directly (`?search=3565`) alongside id/name.
4. `PATCH /workouts/:id` never touches stages or member assignments —
   always use `PUT /workouts/:id/instructions` for stages (resend the full
   array, even to edit just one) and `POST`/`DELETE /workout-assignments`
   for members.
5. `workoutId`/`memberId` on an assignment are permanent — plan the flow as
   "delete + recreate" if you need to move a member to a different program,
   not "edit in place".
6. `durationDays` on the assignment silently falls back to the workout's own
   value if you omit it — send an explicit value only when it should differ
   per-member.
7. Dates (`startDate`/`endDate`) accept plain `"YYYY-MM-DD"` strings — no
   need to send a full timestamp.
8. The list's "Duration" filter dropdown maps to a plain
   `minDurationDays`/`maxDurationDays` range, not a fixed enum — the source
   screens didn't specify preset buckets, see the callout in §2 above.
