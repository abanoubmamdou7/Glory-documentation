# Group classes timetable (mobile)

**New 2026-11-16:** `GET /mobile/group-classes` — the group classes timetable.
Members attend group classes **without booking**, so this is the screen's only
data source. Don't read classes from `GET /mobile/bookings`.

| Screen | Endpoint |
| --- | --- |
| Home → Group classes tab, "تماريني" → group classes | `GET /mobile/group-classes?from=&to=` |
| Private sessions (PT) tab | `GET /mobile/bookings?type=PT` (unchanged) |

## Request

Member bearer token (same as every `/mobile/*` route).

| Query | Format | Default | Notes |
| --- | --- | --- | --- |
| `from` | `YYYY-MM-DD` | today (Amman) | first day, inclusive |
| `to` | `YYYY-MM-DD` | `from` + 6 days | last day, inclusive; at most 62 days after `from` |

Not paginated: one call returns the whole window, sorted soonest first.

## Response

```json
{
  "success": true,
  "data": [
    {
      "id": "cmx...",
      "nameEn": "Yoga",
      "nameAr": "يوغا",
      "date": "2026-11-16",
      "dayOfWeek": 1,
      "dayNameEn": "Monday",
      "dayNameAr": "الاثنين",
      "startTime": "18:00",
      "endTime": "19:00",
      "startAt": "2026-11-16T15:00:00.000Z",
      "endAt": "2026-11-16T16:00:00.000Z",
      "durationMinutes": 60,
      "instructor": { "id": "...", "fullName": "Coach Ahmed", "avatarUrl": null },
      "branch": { "id": "...", "nameEn": "Main branch", "nameAr": "الفرع الرئيسي" },
      "location": "Studio A",
      "capacity": 20,
      "status": "SCHEDULED",
      "cancelReason": null
    }
  ],
  "meta": { "from": "2026-11-16", "to": "2026-11-22", "timezone": "Asia/Amman", "total": 1 }
}
```

- `date`, `dayOfWeek` (0 = Sunday … 6 = Saturday), `startTime` and `endTime`
  are already in **Amman time**. Display them as they come; don't convert
  `startAt` yourself. `startAt`/`endAt` are UTC, for sorting or reminders.
- To group by day, use `date`.
- Nullable: `nameEn`/`nameAr` (falls back to the class title), `endTime`,
  `endAt`, `durationMinutes` (when the class has no end time), `instructor`,
  `branch`, `location`, `capacity`.
- `status`: `SCHEDULED` | `CANCELLED` | `COMPLETED`. **Cancelled classes are
  included** so you can show them crossed out with `cancelReason`. Hide them
  if the design doesn't want them.

## Which classes the member sees

- **Branch:** a member with a home branch sees that branch's classes plus
  classes with no branch. A member with no home branch sees every branch.
- **Package:** a class tied to a package appears only for members holding a
  live (active or frozen) subscription on that package. Classes with no
  package appear for everyone.

So two members can see different timetables for the same week. That is
correct.

## Errors

| Status | When |
| --- | --- |
| `400` | `from`/`to` not `YYYY-MM-DD`, not a real day (e.g. `2026-02-30`), `to` before `from`, or more than 62 days |
| `401` | no member token, or an employee token |

## Where the data comes from

The dashboard's **Scheduling & Classes** screen (`POST /schedule-events`,
type `GROUP_CLASS`, usually monthly recurring). If a class is missing in the
app, first check it was created there as `GROUP_CLASS` (not `PT`), and check
its branch/package against the member's.
