# 06 — My Exercises ("تماريني")

> Audience: **mobile app**. Member Bearer token required on every endpoint
> here. Covers the exercises list, the exercise detail screen, and the
> "اضافة وزن" (Add Weight) action.
>
> ⚠ **Scope note:** creating a Workout program, adding its stages/videos, and
> assigning it to a member are all **dashboard-side** actions — see
> [../frontend/06-workouts.md](../frontend/06-workouts.md)
> (`POST /workouts`, `PUT /workouts/:id/instructions`,
> `POST /workout-assignments`). This module only covers what the member
> does with an assignment that already exists: view it and log their own
> weight. There is no create/edit endpoint for workouts or assignments here.

| # | Method & path | Screen |
| --- | --- | --- |
| 1 | `GET /mobile/workouts` | "تماريني" — the exercises list |
| 2 | `GET /mobile/workouts/:id` | "تفاصيل التمرين" — exercise detail |
| 3 | `GET /mobile/workouts/:id/videos` | Flattened video list (a "watch all" view) |
| 4 | `POST /mobile/workouts/:id/weight` | "اضافة وزن ( للتمرين )" |

---

## 1. `GET /mobile/workouts` — "تماريني"

Query: `page`, `limit`, `status` (`COMPLETED`|`IN_PROGRESS`|`UPCOMING`),
`sortOrder` (default `desc` — newest-assigned first).

```json
{ "id": "...", "status": "IN_PROGRESS",
  "workout": { "id": "...", "nameEn": "Cardio Program", "nameAr": "برنامج كارديو",
               "type": "CARDIO", "level": "BEGINNER" },
  "startDate": "2026-05-01T00:00:00.000Z",
  "endDate": "2026-06-10T00:00:00.000Z",
  "durationDays": 40,
  "instructor": { "id": "...", "fullName": "احمد حسام محمد", "avatarUrl": null },
  "previewVideoUrl": "https://player.vimeo.com/video/...",
  "previewThumbnailUrl": "https://i.vimeocdn.com/..." }
```

Card mapping: status badge — green **"تمرين مكتمل"** = `COMPLETED`, yellow
**"تمرين جاري"** = `IN_PROGRESS`, red **"تمرين قادم"** = `UPCOMING`;
`previewVideoUrl`/`previewThumbnailUrl` = the video thumbnail on the card
(taken from the exercise's first stage); "اسم التمرين" = `workout.nameAr`/
`nameEn`; "نوع التمرين" = localize `workout.type`; "الوقت" = `durationDays`
+ " يوم"; "تاريخ البداية" = `startDate`.

> For `UPCOMING` assignments, `startDate`/`endDate` are `null` (the design
> shows no start-date row on that card, and instead shows **"اصدرت بواسطة"**
> = `instructor.fullName`) — this falls out naturally from the data, no
> special-casing needed on your end beyond "hide the field if null".

## 2. `GET /mobile/workouts/:id` — "تفاصيل التمرين"

```json
{ "id": "...", "status": "IN_PROGRESS",
  "startDate": "2026-05-01T00:00:00.000Z",
  "endDate": "2026-06-10T00:00:00.000Z",
  "durationDays": 40,
  "remainingDays": 22,
  "suggestedWeight": "10.00", "suggestedWeightLast": null,
  "userWeight": null, "userWeightLast": null,
  "canAddWeight": true,
  "instructor": { "id": "...", "fullName": "احمد حسام محمد", "avatarUrl": null },
  "workout": { "id": "...", "nameEn": "Cardio Program", "nameAr": "...",
               "type": "CARDIO", "level": "BEGINNER", "durationDays": 40 },
  "instructions": [
    { "id": "...", "stepNumber": 1, "instructionEn": "Cardio", "instructionAr": "كارديو",
      "videos": [ { "id": "...", "videoUrl": "https://player.vimeo.com/video/...",
                     "thumbnailUrl": "...", "duration": "02:10" } ] },
    { "id": "...", "stepNumber": 2, "instructionEn": "Cardio", "instructionAr": "كارديو",
      "videos": [ { "id": "...", "videoUrl": "...", "thumbnailUrl": "...", "duration": "02:10" } ] }
  ],
  "createdAt": "..." }
```

Mapping: top info grid = `startDate`/`workout.type`/`durationDays`/`endDate`;
**"عدد الايام المتبقية"** (only shown while `IN_PROGRESS`) = `remainingDays`
(computed server-side from `endDate`, recalculated every call — don't cache
it across days); **"تعليمات التمرين العامة"** section = `instructions[]`,
ordered by `stepNumber` ("المرحلة الاولة"/"المرحلة الثانية"/...); each
stage's video box = `instructions[i].videos[0]`. The "نوع التمرين" label
repeated on each stage card is just the parent `workout.type` shown again,
not a distinct per-stage value — instructions in this design don't carry
their own type.

### The weight fields — read this before wiring the UI

The design shows up to **two** weight numbers on a stage card at once, and
the label next to the second number changes between screens (**"الوزن
السابق"**, or a value that looks like **"الوزن اللعب"** in a couple of the
provided mockups — the exact wording wasn't fully legible in the source
screenshots). Based on the schema and how the flow behaves, the intended
mapping is:

| Field | Meaning | Shown as |
| --- | --- | --- |
| `suggestedWeight` | The current target weight for this cycle (set by the coach/system) | **"الوزن المقترح"** — always shown |
| `userWeight` | The weight the member most recently logged via "اضافة وزن" | The second number, once the member has logged at least once |
| `userWeightLast` | The weight the member logged the time before that | Shown once the member has logged **twice** |
| `suggestedWeightLast` | The previous cycle's suggested target | Not obviously mapped to any single label in the screens — exposed for completeness |

> ⚠ **This label mapping is an inference, not a confirmed spec** — the
> screenshots' Arabic labels for the second weight number were ambiguous/
> possibly truncated across the different mockup states. Verify against the
> actual Figma text layers before shipping, and flag back if the intended
> meaning differs — the API fields themselves (`suggestedWeight`/
> `suggestedWeightLast`/`userWeight`/`userWeightLast`) are stable regardless
> of which label each one ends up under.

`canAddWeight` = `true` only while `status === "IN_PROGRESS"` — drive the
"اضافة وزن" button's enabled state from this flag directly rather than
re-deriving the rule client-side.

## 3. `GET /mobile/workouts/:id/videos`

A flattened, stage-ordered list of every video attached to the exercise —
handy for a dedicated "watch all" screen instead of scrolling the full
detail page:

```json
[
  { "id": "...", "videoUrl": "https://player.vimeo.com/video/111",
    "thumbnailUrl": "...", "duration": "01:00",
    "stepNumber": 1, "instructionEn": "Cardio", "instructionAr": "كارديو" },
  { "id": "...", "videoUrl": "https://player.vimeo.com/video/222",
    "thumbnailUrl": "...", "duration": "02:00",
    "stepNumber": 2, "instructionEn": "Cardio", "instructionAr": "كارديو" }
]
```

Same ownership rules as everything else in this module — `404` if the
assignment isn't the caller's.

## 4. `POST /mobile/workouts/:id/weight` — "اضافة وزن"

Body: `{ "weight": "20.00" }` (decimal string, 2 dp — same convention as
every money/weight field in this API, never a JS number).

**201:** the full updated exercise detail (same shape as `GET /:id`) —
`userWeightLast` becomes whatever `userWeight` was before this call, and
`userWeight` becomes the new value. Route the member back to the details
screen and just re-render from this response.

**400:**
- `"You can only add weight while the exercise is in progress"` — the
  button should already be disabled per `canAddWeight`; this is the
  server-side backstop, same pattern as booking cancel/check-in.
- `weight is not a valid decimal number.` — validation failure (matches the
  "قم بإدخال الوزن المقترح الخاص بك لهذه التمرين" placeholder's numeric
  input).

**404:** the assignment doesn't exist or isn't the caller's.

---

## Gotchas checklist

1. No create/edit endpoints exist for workouts or assignments — everything
   here is read-only except the one weight-logging action.
2. `remainingDays` is computed fresh on every request from `endDate` — don't
   cache it locally across app sessions.
3. The weight-label mapping above is a documented inference, not confirmed
   design text — double check before final copy.
4. `startDate`/`endDate` are `null` for `UPCOMING` assignments by design —
   hide those rows rather than showing an empty/zero date.
5. Video URLs are Vimeo embeddable player URLs (see
   [05-bookings-checkin.md](05-bookings-checkin.md) and
   [../frontend/04-video-library.md](../frontend/04-video-library.md) for
   the same convention) — use them in an `<iframe>`/native video player, not
   as a link-out.
