# Unfreeze · Check-out · Sandy plans · Coaching cycle · Onboarding answers

**Delivered 2026-09-28**, from the frontend's
`glory-gym-MISSING-live-routes.postman_collection.json` (a diff of every route
`lib/api` calls against the live `docs-json`). All six missing paths are now
live, at exactly the URLs the collection asked for.

| The collection asked for | Status |
| --- | --- |
| `POST /subscriptions/{id}/unfreeze` | built |
| `POST /checkins/{id}/check-out` | built — **only** this path (see below) |
| `GET /sandy-plans`, `POST /sandy-plans/generate`, `PATCH /sandy-plans/{id}`, `POST /sandy-plans/{id}/approve`, `POST /sandy-plans/{id}/reject` | built |
| `GET /coaching-cycles/{memberId}` | built |
| `GET /members/{id}/onboarding-answers` | built |

Full reference: `docs/API.md` → *Subscriptions*, *Check-ins (history…)*,
*Members*, *Sandy coaching plans*.

---

## 1. Unfreeze a sale

```
POST /subscriptions/{id}/unfreeze     (subscriptions.manage)
{ "reason": "Member returned early" }        // reason is REQUIRED, min 3 chars
```

- `200` returns the subscription in the usual shape, `status: "ACTIVE"`, with
  the recalculated `endDate`.
- **The member gets their unused days back.** Freezing pushed `endDate` out by
  the whole planned span; unfreezing pulls it back by `planned − actual`. Froze
  14 days, came back after 10 → the sale ends 10 days later than before the
  freeze, not 14. Re-render `endDate` and `remainingDays` from the response
  rather than recomputing locally.
- Optional `endedAt` (ISO) to backdate. It must sit **inside** the freeze
  window: before the start or after the planned end → `400`.
- `409` when the sale is not `FROZEN`, or has no open freeze (e.g. a second
  click). `400` when `reason` is shorter than 3 characters.
- `GET /subscriptions/{id}/history` gained a **`UNFREEZE`** row type alongside
  `FREEZE`/`TRANSFER`/`UPGRADE`/`RENEWAL` — the freeze and the unfreeze are two
  rows, not one row that changes. `?type=UNFREEZE` filters on it.
- As the collection asked: status is never changed through `PATCH /{id}` (still
  clerical-only, and it still has no `status` field).

## 2. Check-out

```
POST /checkins/{id}/check-out          (checkins.manage)
{}                                      // body may be empty -> "now"
```

- Only this path was built. The fallbacks in `checkinsApi.checkOut`
  (`/checkout`, `PATCH /checkins/{id}`) are **not** implemented and will keep
  404ing — drop them from the client.
- `200` returns the check-in row with `checkOutAt` and `checkedOutBy`.
  `checkOutAt: null` means the member is **still inside** — that is how you
  render an "open" visit.
- `409` already checked out (a double-tap must not move a recorded departure)
  or reversed; `400` when `checkOutAt` is before the check-in or more than 5
  minutes in the future; `404` unknown.
- `DELETE /checkins/{id}` is unchanged and still means **reversal** ("this
  visit never happened"). A reversed row cannot be checked out.
- `checkOutAt` is now in the list, the detail and the CSV export (new
  `Checked Out At` column), so an "inside now" filter is `checkOutAt === null`
  client-side.

## 3. Coaching cycle

```
GET /coaching-cycles/{memberId}        (workouts.manage OR members.read)
```

```json
{ "memberId": "...", "lastInBodyAt": "...", "lastBodyRecordId": "...",
  "startsOn": "...", "endsOn": "...", "nextInBodyOn": "...",
  "status": "ACTIVE", "daysRemaining": 12,
  "hasApprovedPlan": false, "hasOpenDraft": true }
```

- `status`: `NO_INBODY` (never tested) · `ACTIVE` (inside the window) · `DUE`
  (the next InBody is overdue).
- Always `200` for an existing member — render the empty state from `status`,
  not from a 404. Only an unknown member 404s.
- ⚠ **Policy inferences, flagged so you can correct them:** the cycle is one
  month, anchored on the last InBody (falling back to membership start, then
  registration date). A lapsed cycle is **not** rolled forward — an overdue
  member shows `DUE` with `nextInBodyOn` in the past, on purpose. Tell us if
  the gym's real rule differs.

## 4. Sandy plans

Reads need `workouts.manage` **or** `members.read`; **every write needs
`workouts.manage`**. No new permission keys — the catalog is still 120.

```
GET   /sandy-plans?memberId=&instructorId=&status=DRAFT|APPROVED|REJECTED
POST  /sandy-plans/generate      { memberId, instructorId?, focus? }
GET   /sandy-plans/{id}
PATCH /sandy-plans/{id}          { nameEn?, nameAr?, notes?, workoutType?, level?, instructorId?, exercises? }
POST  /sandy-plans/{id}/approve  {}
POST  /sandy-plans/{id}/reject   { reason }          // min 3 chars
```

The plan object:

```json
{ "id": "...", "memberId": "...", "member": { "id": "...", "memberCode": "...", "fullName": "..." },
  "instructorId": "...", "instructor": { "id": "...", "fullName": "..." },
  "status": "DRAFT",
  "nameEn": "Balanced Full Body Strength", "nameAr": "قوة متوازنة لكامل الجسم",
  "notes": "for the COACH: what the draft was based on and what to watch",
  "workoutType": "STRENGTH_TRAINING", "level": "BEGINNER",
  "exercises": [ { "title": "Goblet Squat", "instructionsEn": "...", "instructionsAr": "...",
                   "setsRepsEn": "3 x 10", "durationRestEn": "60s rest" } ],
  "cycleStartsOn": "...", "cycleEndsOn": "...",
  "bodyRecordId": "...", "generatedBy": "gpt-5.6-luna",
  "workoutId": null, "assignmentId": null,
  "approvedAt": null, "approvedBy": null,
  "rejectedAt": null, "rejectedBy": null, "rejectionReason": null,
  "createdBy": { "id": "...", "fullName": "..." }, "createdAt": "...", "updatedAt": "..." }
```

Things to build against:

- **Generation is a real model call — budget ~15-25s.** Show a spinner with a
  cancel, not an optimistic row. It is the only slow endpoint in this set.
- **A DRAFT is invisible to the member.** `workoutId`/`assignmentId` stay
  `null` until approval; approving fills both and is what puts the programme
  in the member's app.
- **One open draft per member per cycle** → `409` on a second `generate`.
  Rejecting frees the cycle; approving closes it. `hasOpenDraft` /
  `hasApprovedPlan` on the coaching cycle tell you which button to show.
- `PATCH` **replaces the whole `exercises` array** — send the full list, not a
  delta. Omitted fields are left untouched.
- `409` on `PATCH`/`approve`/`reject` once the plan is no longer a `DRAFT`;
  the message names the current status.
- `503` from `generate` means the AI provider is down or returned something
  unusable — **nothing was stored**, so a retry is safe.
- `notes` is written for the coach (what the draft was based on, what to
  watch), not for the member. Don't surface it in the member app.

## 5. Onboarding answers

```
GET /members/{id}/onboarding-answers    (members.read)
```

`[{ questionId, questionEn, questionAr, type, value, answeredAt }]` in the
questions' own order. **`value` is already typed** per question kind —
`string` for TEXT/SINGLE_CHOICE, `number`, `boolean`, `string[]` for
MULTI_CHOICE, and `string[]` of file URLs for PHOTO. An empty array means the
member has not filled it in; that is not an error state.

## Gotchas checklist

- [ ] Drop the `/checkout` and `PATCH /checkins/{id}` fallbacks.
- [ ] Read `endDate`/`remainingDays` back from the unfreeze response.
- [ ] Handle `UNFREEZE` in the subscription history table's type switch.
- [ ] `checkOutAt === null` = still inside.
- [ ] Long spinner on plan generation; `409` means a draft already exists —
      take the coach to it.
- [ ] Don't show plan `notes` to members.
