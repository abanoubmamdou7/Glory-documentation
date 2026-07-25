# 06 — Workouts (dashboard)

> Audience: **web dashboard frontend**. Covers building **Workout program
> templates** (name, type, level, stages, videos) and **assigning** them to
> members — the piece that makes an exercise show up in the mobile app's
> "تماريني" ([mobile/06-workouts.md](../mobile/06-workouts.md)). Reads need
> only a valid token; writes need `workouts.manage` (ADMIN bypasses).

No specific Figma screens existed for this yet — this API was built directly
from the `Workout`/`WorkoutInstruction`/`WorkoutInstructionVideo`/
`WorkoutAssignment` schema, following the same list/create/edit patterns as
[04-video-library.md](04-video-library.md) and
[02-team-management.md](02-team-management.md) (single-call nested create,
`PUT` replace-semantics for a sub-collection). If real screens exist later,
this doc should be revisited against them.

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
| 1 | `POST /workouts` | Create a program, optionally with its stages |
| 2 | `GET /workouts` | List programs |
| 3 | `GET /workouts/:id` | Get a program with its stages + videos |
| 4 | `PATCH /workouts/:id` | Update program scalars |
| 5 | `PUT /workouts/:id/instructions` | Replace the whole stage list |
| 6 | `DELETE /workouts/:id` | Delete a program |
| 7 | `POST /workout-assignments` | Assign a program to a member |
| 8 | `GET /workout-assignments` | List assignments (all members) |
| 9 | `GET /workout-assignments/:id` | Get one assignment |
| 10 | `PATCH /workout-assignments/:id` | Update status/dates/weight/instructor |
| 11 | `DELETE /workout-assignments/:id` | Remove an assignment |

---

## 1. `POST /workouts` — create a program

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `nameEn` | string ≤150 | **yes** | |
| `nameAr` | string ≤150 | no | |
| `type` | `STRENGTH_TRAINING`\|`WEIGHT_TRAINING`\|`FLEXIBILITY_MOBILITY`\|`REHABILITATION_LOW_IMPACT`\|`FUNCTIONAL_TRAINING`\|`CARDIO` | **yes** | |
| `level` | `BEGINNER`\|`INTERMEDIATE`\|`MASTER` | **yes** | |
| `durationDays` | int ≥1 | no | e.g. `40` |
| `instructions` | array | no | Stages, created atomically — see below |

Each `instructions[]` item:

```json
{ "stepNumber": 1, "instructionEn": "Cardio", "instructionAr": "كارديو",
  "videoIds": ["<id from POST /video-library>"] }
```

`stepNumber` must be unique within the array (1-based stage order).
`videoIds` is optional per stage and can list more than one video; every id
must already exist in the Video Library or the whole request `400`s.

**201:** the created program, stages included, each stage's videos
**flattened** (not the raw `{ video: {...} }` junction shape):

```json
{ "id": "...", "nameEn": "Cardio Program", "nameAr": "...",
  "type": "CARDIO", "level": "BEGINNER", "durationDays": 40,
  "createdBy": { "id": "...", "fullName": "Super Admin" },
  "_count": { "assignments": 0 },
  "instructions": [
    { "id": "...", "stepNumber": 1, "instructionEn": "Cardio", "instructionAr": "كارديو",
      "videos": [ { "id": "...", "titleEn": "...", "videoUrl": "...", "thumbnailUrl": "...", "duration": "12:10" } ] },
    { "id": "...", "stepNumber": 2, "instructionEn": "Cardio", "instructionAr": "كارديو", "videos": [] }
  ] }
```

## 2. `GET /workouts` — list

Query: `page`, `limit`, `search` (id/nameEn/nameAr), `type`, `level`,
`sortBy` (`nameEn`|`type`|`level`|`createdAt`), `sortOrder`. List items
include `_count.assignments`/`_count.instructions` (not the full stage
breakdown — call `GET /:id` for that).

## 3. `GET /workouts/:id`

Same shape as the create response.

## 4. `PATCH /workouts/:id`

Scalars only (`nameEn`/`nameAr`/`type`/`level`/`durationDays`) — stages are
**not** editable here, use the dedicated endpoint below (same separation
Branches uses for `PATCH` vs `PUT .../images`).

## 5. `PUT /workouts/:id/instructions` — replace stages

```json
{ "instructions": [
  { "stepNumber": 1, "instructionEn": "Warmup", "videoIds": ["..."] },
  { "stepNumber": 2, "instructionEn": "Main Set", "videoIds": ["...", "..."] }
] }
```

**Replaces the entire stage list** — this is how you add, edit, remove, or
reorder stages: just resend the full desired array. Same validation as
create (unique `stepNumber`, all `videoIds` must exist).

## 6. `DELETE /workouts/:id`

**409** if any member is still assigned this program — remove/reassign those
first (`DELETE /workout-assignments/:id`).

---

## 7. `POST /workout-assignments` — assign to a member

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

## 8–9. `GET /workout-assignments` / `GET /workout-assignments/:id`

List: filter `memberId`/`workoutId`/`status`. Both auth-only.

## 10. `PATCH /workout-assignments/:id`

`workoutId`/`memberId` are **not editable** — to reassign, delete and create
a new one instead (this mirrors treating an assignment as a specific
enrollment, not a mutable pointer). Everything else (`instructorId`/
`status`/`startDate`/`endDate`/`durationDays`/`suggestedWeight`) can change.

> **Setting a new `suggestedWeight` shifts the previous value into
> `suggestedWeightLast`** before saving — the exact mirror of what the
> mobile "اضافة وزن" action does to `userWeight`/`userWeightLast`. This is
> how staff periodically update the member's target weight across cycles.

## 11. `DELETE /workout-assignments/:id`

Removes the assignment — the exercise disappears from that member's
"تماريني". No FK dependents, so this is a plain delete (no 409 case).

---

## Gotchas checklist

1. Uploading a video alone does nothing visible to any member — it has to
   be attached to a stage (`videoIds` on create or on the replace-stages
   call) of a `Workout`, and that `Workout` has to be assigned via
   `POST /workout-assignments`.
2. `PATCH /workouts/:id` never touches stages — always use
   `PUT /workouts/:id/instructions` for that, even to edit just one stage
   (resend the full array).
3. `workoutId`/`memberId` on an assignment are permanent — plan the flow as
   "delete + recreate" if you need to move a member to a different program,
   not "edit in place".
4. `durationDays` on the assignment silently falls back to the workout's own
   value if you omit it — send an explicit value only when it should differ
   per-member.
5. Dates (`startDate`/`endDate`) accept plain `"YYYY-MM-DD"` strings — no
   need to send a full timestamp.
