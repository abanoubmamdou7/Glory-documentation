# 05 — Home, Check-in, Bookings, Rating & Notifications

> Audience: **mobile app**. Member Bearer token required on every endpoint
> here. Covers the home screen's three tabs (**المواعيد**, **الحصص
> الجماعية**‑adjacent booking list, **تسجيل للجيم**), the **الحجوزات** tab,
> post-session rating (**تقيم للحصه**), and the **الاشعارات** inbox.
>
> ⚠ **Scope note:** bookings themselves (a PT/class session on the calendar)
> are created by gym staff scheduling — that's a dashboard screen belonging
> to a later phase and is **not built yet**. This module covers everything
> the member does with a booking that already exists: check into it, cancel
> it, and rate it afterward. The "الحصص الجماعية" home tab's workout-video
> card (exercise name/type/duration) is a different feature (Workouts &
> Video Library) and is **out of scope** for this set of APIs.

| # | Method & path | Screen |
| --- | --- | --- |
| 1 | `POST /mobile/checkin/qr` | "تسجيل للجيم" — generate the QR |
| 2 | `GET /mobile/checkin/qr/:id` | Poll while the QR is on screen |
| 3 | `GET /mobile/bookings` | "الحجوزات" / home "اخر الحجوزات" |
| 4 | `GET /mobile/bookings/:id` | Booking detail |
| 5 | `PATCH /mobile/bookings/:id/cancel` | "الغاء الحصة" |
| 6 | `POST /mobile/bookings/:id/check-in` | "تسجيل دخول التدريب" |
| 7 | `GET /mobile/assessment/questions` | "تقيم للحصه" question list |
| 8 | `POST /mobile/bookings/:id/rate` | Submit the rating |
| 9 | `GET /mobile/bookings/:id/rating` | View a submitted rating |
| 10 | `GET /mobile/notifications` | "الاشعارات" list |
| 11 | `GET /mobile/notifications/unread-count` | Bell badge count |
| 12 | `GET /mobile/notifications/:id` | Notification detail modal |
| 13 | `PATCH /mobile/notifications/read-all` | Mark all read |

---

## 1–2. "تسجيل للجيم" — QR gym-entry check-in

This is the **only** flow that uses a QR code + scanner. It is separate from
booking check-in (§6), which is a plain button tap.

```mermaid
sequenceDiagram
  participant App
  participant API
  participant Scanner as Front-desk scanner
  App->>API: POST /mobile/checkin/qr
  API-->>App: {id, token, expiresAt, expiresInSeconds: 15}
  Note over App: render `token` as a QR code; countdown from expiresInSeconds
  loop every 1-2s while shown
    App->>API: GET /mobile/checkin/qr/{id}
    API-->>App: {status: PENDING}
  end
  Scanner->>API: POST /checkin/scan {token}
  API-->>Scanner: member + daysRemaining
  App->>API: GET /mobile/checkin/qr/{id}
  API-->>App: {status: CONSUMED, daysRemaining}
  Note over App: show "لقد تم دخول الجيم بنجاح!" — متبقي {daysRemaining} يوم
```

### `POST /mobile/checkin/qr`

No body. **201:**

```json
{ "id": "cm...", "token": "b6f2...-uuid", "expiresAt": "...", "expiresInSeconds": 15 }
```

- `expiresInSeconds` is configurable server-side (`GYM_QR_TTL_SECONDS`,
  default 15) — **always drive the countdown from the response**, never
  hardcode 15.
- Calling this again **immediately invalidates** any still-open previous
  token from this member (so only one QR is ever scannable at a time — the
  "توليد QR كود اخر" button after expiry, or an auto-refresh, is safe to spam).
- Encode `token` into the QR image client-side (any QR library — the backend
  only needs the raw string back via the scanner).

### `GET /mobile/checkin/qr/:id`

Poll this every ~1–2 seconds while the QR is on screen (`:id` is the `id`
from the generate response, **not** the token).

```json
{
  "id": "cm...",
  "status": "PENDING",
  "expiresAt": "...",
  "consumedAt": null,
  "checkIn": null,
  "daysRemaining": null
}
```

`status` is one of:
- **`PENDING`** — still valid, keep the QR + countdown on screen.
- **`EXPIRED`** — countdown hit zero (or a newer QR was generated). Enable
  "توليد QR كود اخر".
- **`CONSUMED`** — the front desk scanned it. `daysRemaining` is now populated
  — show the success screen: *"لقد تم دخول الجيم بنجاح! أهلاً بك في عائلة
  جلوري جيم! و نود ابلاغك بانه متبقي {daysRemaining} يوم من اشتراكك في
  الجيم"*, then an "الرئيسية" button back home. Stop polling once you see
  this state.

`404` if `:id` doesn't belong to the calling member.

**Business rule to know:** the scan is refused (by the staff-side endpoint,
so this shows up as the QR simply never turning `CONSUMED`) if the member has
**no active subscription** — there is nothing for the app to do here except
let the code expire; the gym needs to sort out the membership first.

---

## 3–4. "الحجوزات" / "اخر الحجوزات" — booking list

### `GET /mobile/bookings`

Query: `page`, `limit`, `status` (`PENDING`|`PAID`|`UNPAID`|`CANCELED`),
`sortOrder` (`desc` default = most recent first; pass `asc` for
soonest-upcoming-first — handy for the home widget).

```json
{ "id": "...", "type": "PT", "status": "PENDING", "channel": "APP",
  "dateTime": "2026-08-01T18:20:00.000Z",
  "checkedInAt": null,
  "package": { "id": "...", "nameEn": "PT Package", "nameAr": "باقة تدريب شخصي", "membershipType": "PT" },
  "branch": { "id": "...", "nameEn": "6th October" },
  "instructor": { "id": "...", "fullName": "احمد حسام", "avatarUrl": null },
  "subscription": { "id": "...", "remainingSessions": 3 },
  "canCancel": true, "canCheckIn": true, "canRate": false
}
```

Card mapping: "اسم الباكدج" = `package.nameAr`/`nameEn`; "تدريب شخصي" badge =
`type === "PT"` (localize other types: `APPOINTMENT` → "موعد", `CLASS` →
"حصة جماعية", `GYM` → "جيم"); "الوقت"/"التاريخ" = split `dateTime`
client-side; "اسم المدرب" = `instructor.fullName`.

**Drive the two buttons directly from the flags** — don't re-derive the
rules client-side:
- **"الغاء الحصة"** enabled ⟺ `canCancel`.
- **"تسجيل دخول التدريب"** enabled ⟺ `canCheckIn` (grey/disabled once
  `checkedInAt` is set or the booking is canceled — matches the design's
  second booking card showing it greyed out).

### `GET /mobile/bookings/:id`

Same shape as a list item, for a detail screen or post-notification refresh.
`404` if the booking isn't the caller's.

---

## 5. "الغاء الحصة"

`PATCH /mobile/bookings/:id/cancel`, no body. **200:** the updated card
(`status: "CANCELED"`, `canCancel`/`canCheckIn` now `false`).

**400:** already canceled, or already checked into (attended sessions can't
be canceled — the button should already be disabled per `canCancel`, this is
the server-side backstop).

> Canceling does **not** itself restore a session to the subscription's
> `remainingSessions` — reconciling that is a sales-lifecycle concern
> (dashboard, not yet built). The booking is simply freed from the member's
> active list.

## 6. "تسجيل دخول التدريب" — one-tap self check-in

`POST /mobile/bookings/:id/check-in`, no body. **Not a QR flow** — the
member is clearly already at the session inside the app, so this is a single
tap with no scanner involved.

**201:**

```json
{ "id": "...", "checkedInAt": "2026-08-01T18:22:00.000Z", "canCheckIn": false, "canRate": true,
  "package": { "nameAr": "باقة تدريب شخصي", "nameEn": "PT Package" },
  "instructor": { "fullName": "احمد حسام" },
  "subscription": { "remainingSessions": 3 }
}
```

Show: *"لقد تم دخولك للحصة بنجاح! أهلاً بك في عائلة جلوري جيم! لقد تم تسجيل
دخول لحصة ({package.nameAr}) مع الكوتش ({instructor.fullName}) متبقي معك
{subscription.remainingSessions} حصص"* → "الرئيسية" button.

**400:** booking was canceled, or already checked in (button should already
be greyed out per `canCheckIn`, same backstop pattern as cancel).

---

## 7–9. "تقيم للحصه" — post-session rating

```mermaid
sequenceDiagram
  participant App
  participant API
  App->>API: GET /mobile/assessment/questions
  API-->>App: [ {id, questionAr, questionEn, sortOrder}, ... ]
  Note over App: render one 1-5 star row per question; "إرسال" disabled until all answered
  App->>API: POST /mobile/bookings/{id}/rate {answers:[{questionId,answer}...]}
  API-->>App: updated booking (canRate:false)
  Note over App: "لقد تم تقيم الحصة بنجاح!" success modal
```

### `GET /mobile/assessment/questions`

Public to the app (member bearer). Returns active questions, ordered:

```json
[ { "id": "...", "questionEn": "How would you rate the coach's performance?",
    "questionAr": "ما مدى تقييمك لأداء المدرب؟", "sortOrder": 1 }, ... ]
```

### `POST /mobile/bookings/:id/rate`

Only allowed once the booking's `canRate` is `true` (checked-in, not yet
rated). Body must answer **every** active question **exactly once**:

```json
{ "answers": [
  { "questionId": "q1", "answer": 5 },
  { "questionId": "q2", "answer": 4 }
] }
```

`answer` is 1–5 (stars). **400** if: the booking isn't checked in yet,
already rated, a question is missing, an extra/unknown `questionId` is sent,
a question is answered twice, or `answer` is outside 1–5.

**201:** the updated booking (same shape as check-in's response) — reuse the
same success-modal copy pattern (package/coach/remaining sessions), since the
design shows it identically to the check-in success screen.

### `GET /mobile/bookings/:id/rating`

View what was submitted: `{ id, type: "GOOD"|"BAD", sendDate, answers: [{ answer, question: {questionEn, questionAr} }] }`.
`type` is computed server-side from the average star rating (≥3 → `GOOD`).
`404` if the booking hasn't been rated.

---

## 10–13. "الاشعارات" — notification inbox

Delivery rows appear here once the gym sends a broadcast from the dashboard
(a later phase) — this module is the member's read/unread inbox view over
those deliveries.

### `GET /mobile/notifications`

Paginated, newest first; `?unreadOnly=true` to filter.

```json
{ "id": "...", "titleEn": "Get 4 months free", "titleAr": "اوفر شهر 4",
  "bodyEn": "...", "bodyAr": "اور شهرين بقيمة 90 دينار", "type": "PUSH",
  "imageUrl": null, "read": false, "readAt": null, "createdAt": "2026-08-01T09:00:00.000Z" }
```

Group cards by date client-side from `createdAt` (اليوم / الامس / an exact
date for older ones, matching the design). The red unread dot = `!read`.

### `GET /mobile/notifications/unread-count`

`{ "count": 3 }` — poll this for the bell badge (e.g. on app resume/home load).

### `GET /mobile/notifications/:id`

Opens the detail modal (avatar, title, full body, "السابق" to dismiss) —
**marks the notification read as a side effect**, so refresh the list/badge
after this call and the red dot disappears.

### `PATCH /mobile/notifications/read-all`

No body. `{ "updated": 2 }` — bulk "mark all read" if the design adds that
control later.
