# Mobile — InBody History

> Screen: **"InBody history"** — every In-Body test the member has taken,
> newest first, with what changed since the previous one, a header summary
> and charts. Added 2026-09-12. No Figma screens existed for this; built
> from the request + the `BodyRecord` model, so treat the layout notes as
> suggestions and the field names as fixed.

Member bearer token (`Authorization: Bearer <memberAccessToken>`) on every
call. Read-only — a test only ever comes from the gym's InBody device
(auto-imported, `source: INBODY`) or from staff typing it on the dashboard
(`source: MANUAL`). Both show up here.

## 1. Screen → endpoint

| UI | Endpoint |
| --- | --- |
| Header card (total tests, first/last date, latest values, "since you started") | `GET /mobile/inbody/summary` |
| Charts (weight / muscle / fat % over time) | `GET /mobile/inbody/trends?limit=12` |
| The list (one card per test, with ▲▼ vs previous) | `GET /mobile/inbody?page=1&limit=20` |
| Filter chips "Device" / "Manual", date range | `GET /mobile/inbody?source=INBODY`, `?dateFrom=2026-01-01&dateTo=2026-03-31` |
| Tap a card → detail | `GET /mobile/inbody/:id` |

The older `GET /mobile/body-records?type=IN_BODY_TEST` still works (same
rows) but returns decimal **strings** and no deltas — use the new endpoints
for this screen.

## 2. The test object

Every list item, `summary.latest` and the detail share one shape:

| Field | Meaning |
| --- | --- |
| `recordedAt` | When the test was taken (device time for imports). |
| `source` | `INBODY` (device) or `MANUAL` (staff). Render a small badge. |
| `device` | Device model, e.g. `InBody770`; `null` for manual. |
| `enteredBy` | `{ id, fullName }` of the staff member for manual entries; `null` for device imports. |
| `pdfUrl` | The result sheet if one was attached; usually `null`. |
| `metrics` | `weight, muscleMass, bodyFat, bodyWater, visceralFat, bmi, bmr, metabolicAge` — **numbers**, `null` when not measured. |
| `extras` | `inbodyScore, height, bodyFatMass` — from the device payload; all `null` for manual entries. |
| `changes` | This test minus the test taken **before it** (chronologically — not "the next card in the list", though on a newest-first list they coincide). `{ previousId, previousAt, <metric>: { previous, delta } }`; `null` for the member's first test. `delta` is 2-dp and `null` when either side is missing. |

Sign convention: `delta < 0` means the value went **down** since last time —
good for weight/fat, bad for muscle. Colour per metric, not per sign.

## 3. Endpoints

### `GET /mobile/inbody`
Paginated, newest first. Query: `page`, `limit`, `source`, `dateFrom`,
`dateTo` (YYYY-MM-DD, UTC days, `dateTo` inclusive). Response
`{ success, data: Test[], meta: { page, limit, total, totalPages } }`.

### `GET /mobile/inbody/summary`
```json
{ "total": 3, "bySource": { "inbody": 1, "manual": 2 },
  "firstAt": "2026-06-01T08:00:00.000Z", "latestAt": "2026-09-12T10:00:00.000Z",
  "latest": { …Test… },
  "sinceFirst": { "weight": { "previous": 84, "delta": -3.5 }, "…": {} } }
```
`latest` / `sinceFirst` are `null` with 0 / <2 tests — render the empty
state from `total === 0`.

### `GET /mobile/inbody/trends?limit=12`
`{ metrics: ["weight", …8 names], points: [{ id, recordedAt, source, weight, muscleMass, … }] }`
— **oldest → newest**, the last `limit` tests (2–60). One call feeds every
chart; skip `null` points per metric rather than drawing them as 0.

### `GET /mobile/inbody/:id`
The Test object plus `raw`: the device's full payload verbatim for
`INBODY` tests (keys like `WT`, `SMM`, `PBF`, segmental values… — whatever
the LookinBody API returned), `null` for manual ones. Show it as a
"More from the device" section if you want; the eight `metrics` are the
guaranteed part.

## 4. Errors & gotchas

- `404` on `/:id` for another member's test, a body-*measurement* record,
  or an unknown id — never leaks whether it exists.
- `401` for an employee (dashboard) token, like every `/mobile/*` route.
- `400` on a malformed date or `limit` outside 2–60.
- A new device test arrives **asynchronously** (webhook from InBody) and
  also raises an in-app notification — refresh the list on that
  notification (`GET /mobile/notifications`, title "Your InBody results are
  ready") or on screen focus; don't assume the row exists the second the
  member steps off the device.
- `metabolicAge` is only filled for manual entries (InBody devices do not
  report it).
