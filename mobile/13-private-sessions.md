# Private sessions (PT) — booking, QR check-in, rating

**The member's whole private-training journey in the app**, end to end. The
QR part is new (2026-09-30): at the studio the **coach shows a code and the
member scans it** to mark themselves present.

> ⚠ **Two different QR codes exist in this app — do not mix them up.**
>
> | | Gym entry | Private session |
> | --- | --- | --- |
> | Who shows the code | **the member** (`POST /mobile/checkin/qr`) | **the coach** |
> | Who scans it | the front desk | **the member** (`POST /mobile/session-checkin/scan`) |
> | Produces | a `GYM` check-in | attendance on **that PT booking** |
> | Lives for | ~15 seconds | ~5 minutes |
>
> They are opposite directions on purpose: at the desk the member is the one
> holding a screen; in the PT studio the coach is.

---

## 1. The screens

| # | Screen | Endpoint |
| --- | --- | --- |
| 1 | "حصصي الخاصة" — my sessions | `GET /mobile/bookings?status=` (each card has `type: "PT"`) |
| 2 | Session detail | `GET /mobile/bookings/:id` |
| 3 | **Scan to check in** | `POST /mobile/session-checkin/scan` |
| 4 | Check in without a code (fallback) | `POST /mobile/bookings/:id/check-in` |

⚠ **The coach can also mark the session attended from their own screen** (new
2026-10-15, no code involved). So a booking can become `attended` without the
member doing anything — refresh the card when the screen regains focus, and
treat an "already checked in" error on the button or the scan as success rather
than a failure. It usually means the coach got there first.
| 5 | Cancel | `PATCH /mobile/bookings/:id/cancel` |
| 6 | Rate the session | `POST /mobile/bookings/:id/rate` |
| 7 | See a past rating | `GET /mobile/bookings/:id/rating` |
| 8 | Chat with the PT coach | `GET /mobile/chat/conversations` → the thread whose `coachTypes` contains `"PT"` |

A private session is a **booking with `type: "PT"`**. Everything on the
bookings screens already works for it; the sections below only cover what is
specific to private training.

## 2. Scanning the coach's code

```
POST /mobile/session-checkin/scan     (member bearer)
{ "token": "<the string decoded from the QR image>" }
```

`200` returns **the updated booking card** — the exact same object the
one-tap check-in returns, so the screen can reuse its success state:

```json
{ "id": "...", "type": "PT", "checkedInAt": "2026-10-01T17:02:11.000Z",
  "canCheckIn": false, "canRate": true,
  "package": { "nameAr": "باقة تدريب شخصي", "nameEn": "PT Package" },
  "instructor": { "fullName": "الكوتش سارة" },
  "subscription": { "remainingSessions": 3 } }
```

Show: *"تم تسجيل دخولك للحصة! مع الكوتش {instructor.fullName} — متبقي
{subscription.remainingSessions} حصص"* → then the **rate** action.

### Errors, and what to say

| Status | When | Suggested copy |
| --- | --- | --- |
| `404` | the code isn't a real session code | "كود غير صالح — اطلب من الكوتش يعرض الكود من جديد" |
| `400` "already used" | someone already scanned it | "الكود ده اتستخدم — لو ما سجّلتش دخولك كلّم الكوتش" |
| `400` "expired" | the code timed out (~5 min) | "الكود انتهت صلاحيته — اطلب من الكوتش يعرضه تاني" |
| `403` another member's session | scanned the wrong screen | "الحصة دي مش بتاعتك" |
| `400` canceled / already attended | the booking itself | reuse the existing booking-card copy |
| `403` outstanding balance | the member owes money | show the message as-is — it points them at the front desk |
| `403` "check in at reception first" | no gym entry today (2026-11-03) | "لازم تسجّل دخولك من الرسبشن الأول قبل الحصة" |

⚠ **Reception first (2026-11-03).** Both the scan and the in-app button refuse
a PT session until the member has checked in at reception (the gym QR) **the same
day**. A refused scan does not burn the code — after the gym entry the member
scans the same code again. Consider prompting "سجّل دخول الجيم الأول" with a
link to the gym QR screen.

⚠ **The scan is not a bypass.** It runs the *same* check-in as the button, so
every rule still applies — including the gym's outstanding-balance policy. A
refused scan **does not burn the code**: the member can settle up at the desk
and scan the same code again while it is still valid.

### Client notes

- **Point the camera at the coach's screen, not at a printed sheet.** Each
  code belongs to one session and dies in ~5 minutes, so nothing is worth
  caching or screenshotting.
- Re-showing the code on the coach's side **invalidates the previous one** —
  if a scan says "expired"/"already used" right after the coach re-tapped,
  just scan the new image.
- Treat the response exactly like the button's: same card, same next step.
- Keep the button (`POST /mobile/bookings/:id/check-in`) in the UI as a
  fallback for a dead phone/screen — it is unchanged and still works.

## 3. What the coach sees (for context — not mobile endpoints)

The coach portal calls **`POST /coach-sessions`** to book the session in the
first place, `GET /coach-sessions?date=` for the day's sessions,
`POST /coach-sessions/:bookingId/qr` to put a code on screen, and polls
`GET /coach-sessions/:bookingId/qr/:tokenId` until it flips to `CONSUMED` —
which is the coach's "✓ checked in" moment. Useful to know when you are
demoing the two devices side by side.

**Where a private session comes from** *(2026-10-01)*: the coach (or the front
desk) books it. There is **no endpoint for a member to book their own PT
session** — the app lists and attends sessions, it does not create them. So if
a member asks for a session in the app, that is a conversation with the coach
(the PT chat thread), not a booking call. A freshly booked session appears in
`GET /mobile/bookings` straight away with `status: PAID` when the member's PT
subscription covers it, or `PENDING` for a trial/complimentary one.

A ready-to-run Postman file for the whole flow (book → show QR → scan →
confirm, coach and member tokens both wired) ships as
`postman/GloryGym-PrivateSessions.postman_collection.json`.

## 4. Rating, and the coach chat

Rating is unchanged: after `checkedInAt` is set, `canRate` becomes true and
`POST /mobile/bookings/:id/rate` takes **one answer per active question**
from `GET /mobile/assessment/questions` — none missing, none extra.

The PT coach may be a **different person** from the gym coach (since
2026-09-29). The member therefore has up to **two chat threads**; pick the
PT one by `coachTypes` containing `"PT"`. See
[08-chat.md](08-chat.md).

## Gotchas checklist

- [ ] Don't reuse the gym-entry QR screen — this one is a **scanner**, not a
      code display.
- [ ] Handle all five scan errors; three of them are recoverable on the spot.
- [ ] Don't cache or screenshot session codes.
- [ ] A refused scan (unpaid balance) leaves the code usable — offer "retry".
- [ ] Keep the one-tap check-in as a fallback.
