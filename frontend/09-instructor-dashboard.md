# 09 — Instructor Dashboard (Home / Reservations)

> Audience: **web dashboard frontend**. Covers the instructor's own
> **"Dashboard"** landing screen — a **separate dashboard surface from the
> admin one**, with its own sidebar (Dashboard, Members, Coach Chat,
> Reports & Analytics, Workouts, Follow-up Programs, Settings). Reads need
> only a valid token; **Create Reservation** needs `reservations.manage`
> (ADMIN bypasses).

⚠ **Model note:** this screen is backed by the `Reservation` model —
**not** `Booking` (the mobile app's own PT/class flow, already built) and
**not** `ScheduleEvent` (a different calendar concept). The blueprint
explicitly calls out these three as an intentional overlap to keep
separate; don't conflate them when wiring this screen.

| # | Method & path | Screen element |
| --- | --- | --- |
| 1 | `GET /instructor-dashboard/summary` | The 3 stat cards (Total Schedule / PT Sessions / Assessment) |
| 2 | `GET /instructor-dashboard/calendar` | The month/week/day calendar grid's per-day pills |
| 3 | `GET /instructor-dashboard/reservations` | The day-detail side panel (click a calendar day) |
| 4 | `POST /instructor-dashboard/reservations` | "+ Create Reservations" |
| 5 | `GET /instructor-dashboard/members` | The "Total Members" table at the bottom |

---

## Scoping — read this first

- Endpoints **1–4** are scoped to **the calling employee's own id** — this
  is a personal "my schedule" view (matches "Good Morning, Ahmed Hossam").
  An `INSTRUCTOR` sees only reservations where they're the assigned coach;
  a `STAFF`/`ADMIN` calling these directly would see their own (likely
  empty) schedule — this screen is meant to be used *as* the instructor.
- Endpoint **5** (`members`) is **gym-wide**, not instructor-scoped — it's
  a proactive-renewal-outreach tool ("who's running low on days"), matches
  the mockup's large "Total Members: 2080" figure which is clearly the
  whole gym, not one coach's personal roster.

## 1. `GET /instructor-dashboard/summary` — the 3 stat cards

Query: `view` (`day`|`week`|`month`, default `month` — matches the
Day/Week/Month toggle), `date` (anchor date within the range, default
today).

```json
{ "totalSchedule": 8, "ptSessions": 3, "assessment": 5,
  "range": { "view": "month", "from": "2026-02-01T00:00:00.000Z", "to": "2026-03-01T00:00:00.000Z" } }
```

`totalSchedule` = all reservation types combined in the range (including
`PACKAGE`, which has no dedicated card in the mockup). Re-call this
whenever the Day/Week/Month toggle or the visible month/week/day changes.

## 2. `GET /instructor-dashboard/calendar` — the grid

Same `view`/`date` query. Returns **only days that have reservations** —
don't iterate every day of the month client-side, just look up by date:

```json
[
  { "date": "2026-02-11", "total": 3,
    "preview": [
      { "id": "...", "type": "PT_SESSION", "memberName": "Ahmed Hossam" },
      { "id": "...", "type": "ASSESSMENT", "memberName": "Sara Ali" },
      { "id": "...", "type": "ASSESSMENT", "memberName": "Omar Nabil" }
    ] }
]
```

Card mapping: each `preview` entry is one pill on that day's cell
(orange-bordered for `PT_SESSION`, green-bordered for
`ASSESSMENT`/`PACKAGE` — matches the two pill colors shown). `preview` is
capped at 3 — if `total > 3`, show a "+N more" affordance and let the user
click into the day for the full list (endpoint 3). Clicking **any** day
(including ones with 0 reservations) opens the side panel.

## 3. `GET /instructor-dashboard/reservations` — the day-detail panel

Query: `date` (**required**). Fires when a calendar day is clicked.

```json
[
  { "id": "...", "type": "PT_SESSION",
    "scheduledAt": "2026-02-20T18:20:00.000Z",
    "suggestedWeight": null,
    "member": { "id": "...", "fullName": "Ahmed Hossam", "avatarUrl": null },
    "package": { "id": "...", "nameEn": "PT 12 Sessions", "nameAr": "باقة تدريب شخصي", "membershipType": "PT" },
    "branch": null }
]
```

Panel header = the clicked date + `"{count} Reservations"`. Card mapping:
"Package Name" = `package.nameEn`/`nameAr` (or a generic label if `package`
is `null` — a reservation isn't required to have one, e.g. a bare
assessment slot); the green "PT Sessions" badge = localized `type`; "Member
Name" = `member.fullName`; "Date"/"Time" = split `scheduledAt` client-side;
"Member Details" button = navigate using `member.id` to
`GET /members/:id` ([11-members.md](11-members.md), the "Members" nav
item's profile screen).

**Empty array** → render the empty state exactly as designed: calendar
icon, **"No Reservations For This Day."**, and the same
**"+ Create Reservations"** button as the header (both call endpoint 4).

## 4. `POST /instructor-dashboard/reservations` — "+ Create Reservations"

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `type` | `ASSESSMENT`\|`PT_SESSION`\|`PACKAGE` | ✅ | |
| `memberId` | id | ✅ | |
| `packageId` | id | no | "Package Name" on the resulting card |
| `instructorId` | id | conditional | Omit when the logged-in user **is** the instructor — it defaults to them. **Required** when a `STAFF`/`ADMIN` is booking on a coach's behalf |
| `branchId` | id | no | |
| `scheduledAt` | ISO datetime | ✅ | Full date **and** time — both the calendar day and the card's Date/Time |
| `suggestedWeight` | decimal string | no | PT-specific |

**201:** the created reservation (same shape as a day-detail card) — close
the modal and refresh the calendar/day-detail/summary for the affected day.

**Errors:** `400` unknown `memberId`/`packageId`/`branchId`, an
`instructorId` that isn't a real `INSTRUCTOR`, or a missing `instructorId`
when the caller isn't one themselves; `403` missing `reservations.manage`.

## 5. `GET /instructor-dashboard/members` — the members table

Query: standard pagination (`page`/`limit`) + `search` (matches the design's
**"Search by Member (ID, Name, Phone Number or E-mail)"** placeholder
exactly) + `sortOrder` (default `asc` = soonest-expiring first).

```json
{ "id": "...", "memberCode": "45627", "fullName": "Dwight Rohan",
  "email": "Reyna_King51@hotmail.com", "avatarUrl": null,
  "phone": "501234567", "phoneCountryCode": "+966", "remainingDays": 10 }
```

Column mapping: "ID Number" = `memberCode`; "Members Details" = avatar +
`fullName` + `email`; "Mobile Number" = flag (derive from
`phoneCountryCode`) + `phoneCountryCode` + `phone`; "Remaining Days" =
`remainingDays` + `" Days"`; "Action" (eye icon) → `GET /members/:id`
([11-members.md](11-members.md)), same as "Member Details" above.

Only members with a **currently-active subscription** are returned (a
member with none has no meaningful "remaining days" for this widget).

> ⚠ **Documented UI mismatch, not built:** the mockup shows a
> "Remaining Assessment" panel directly above this table with "Workout
> Type / Level / Duration" filters and unrelated copy ("Total and scheduled
> posts per social platform", "82 new posts scheduled") that reads like a
> copy-pasted template block, not real gym-app content. The
> Workout-Type/Level filters it shows already have a real backing endpoint
> — `GET /workouts?type=&level=` (see
> [06-workouts.md](06-workouts.md)) — if this section turns out to be a
> genuine "browse workout templates" widget once real copy/design exists,
> wire it to that existing endpoint rather than building something new.

---

## Other buttons on this screen (no new API)

- **"+ Add New Workout"** (top of the page, separate from "Create
  Reservations") → the "Workouts" nav item's own list + 3-step wizard, now
  fully documented in [06-workouts.md](06-workouts.md) (confirmed against
  its own real Figma screens).
- **Language toggle ("Ar")**, notification bell, avatar menu → out of
  scope for this doc (standard dashboard chrome).

---

## Gotchas checklist

1. Endpoints 1–4 are **personal** (the caller's own reservations); endpoint
   5 is **gym-wide**. Don't assume one scoping rule applies to the whole page.
2. `GET .../reservations` (day-detail) requires `date` — there's no
   "show me everything" mode; always pair it with a clicked calendar day.
3. `package` can be `null` on a reservation card — not every reservation
   type needs one (e.g. a plain assessment slot).
4. Creating on behalf of another instructor (`STAFF`/`ADMIN` caller) needs
   an explicit `instructorId` — omitting it 400s with a clear message
   rather than silently guessing.
5. The "Remaining Assessment" block's filters/copy in the mockup don't
   match this app — see the callout in §5 above before building it.
