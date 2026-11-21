# Marking a private session attended (coach screens)

**New 2026-10-15:** `POST /coach-sessions/:bookingId/check-in` — the coach
marks the member present themselves, with no QR and nothing from the member's
phone.

## ⚠ Reception first (2026-11-03)

**A PT session can only be marked attended after the member checked in at
reception the same day** (gate QR or desk, Amman calendar day). This applies to
all three options below and to the desk's own `POST /checkins`. Without it the
call is **`403`** "This member has no reception check-in today…" — show it
as-is and send the member to reception.

**Admin exception:** an ADMIN may mark it anyway by sending a written reason:

```json
POST /coach-sessions/:bookingId/check-in
{ "gymEntryOverrideReason": "Reception scanner was down" }
```

≥ 3 characters, stored on the check-in notes. For an admin, show a reason box
when the `403` message mentions `gymEntryOverrideReason`; for a coach or staff
there is no override — don't show the box.

## Three ways a PT session gets marked attended

| # | Who acts | Endpoint | Use it when |
| --- | --- | --- | --- |
| 1 | **the coach** | `POST /coach-sessions/:bookingId/check-in` | the member is in front of you — the normal case |
| 2 | the member scans the coach's code | coach: `POST /coach-sessions/:bookingId/qr` → member: `POST /mobile/session-checkin/scan` | you want the member's own device to confirm it |
| 3 | the member taps in their app | `POST /mobile/bookings/:id/check-in` | the member marks their own attendance |

All three run the **same** check-in underneath, so the subscription link, the
outstanding-balance rule and the session consumption are identical. Only the
record differs: option 1 is stored as `source: STAFF` with the coach as the
creator, options 2 and 3 as `source: MOBILE`.

## The screen

`GET /coach-sessions?date=YYYY-MM-DD` gives the coach's day. Each row already
carries `attended` and `openQr`, so one list drives everything:

- `attended: false`, `openQr: null` → show **both** actions: a primary
  "تحضير" (check in) button calling option 1, and a secondary "إظهار الكود"
  for option 2.
- `attended: false`, `openQr: { id, expiresAt }` → a code is live on screen;
  keep polling `GET /coach-sessions/:bookingId/qr/:tokenId`. The manual button
  stays available — if the member's phone is flat, the coach just taps it, and
  **the live code is retired automatically**.
- `attended: true` → show it as done with the time. **Disable both buttons.**

## Once per session — and make the UI say so

A second call is **`409 This session is already checked in`**. It is not a
silent success on purpose: a PT check-in **consumes a session** from the
member's subscription, so a quiet repeat could spend two.

What that means for the screen:

1. **Disable the button the moment you fire the request** and keep it disabled
   on success. Don't rely on the server to be the only guard — it will refuse,
   but the coach shouldn't see an error they could have been spared.
2. **Treat `409` as "already done", not as a failure.** Refresh the row and
   show the attended state; don't show a red error. The likeliest cause is a
   double-tap or the member having scanned the QR a second earlier.
3. Concurrent taps cannot both succeed — one gets `200`, the rest `409` — so
   you can never double-consume a session by mashing the button.

## Errors

| Status | Meaning | What to show |
| --- | --- | --- |
| `403` | not this session's coach, and no `checkins.manage` | hide the button for sessions that aren't theirs |
| `403` | no reception check-in today | "العضو لازم يسجّل دخول من الرسبشن الأول" — admins get a reason box (above) |
| `404` | unknown session | refresh the day |
| `409` | already checked in | the attended state (see above) |
| `409` | not a PT booking / canceled | shouldn't be reachable from this list |
| `409` | the PT subscription has no sessions left | "الحصص خلصت" — the desk needs to renew or add sessions |
| `403` | the member owes money and the policy is `BLOCK` | show the message as-is: it says the front desk can check them in and record the approval. **A coach has no override** — that is deliberate, so the approval is recorded by whoever authorised it |

## Two limits worth knowing

⚠ **One appointment check-in per member per calendar day.** A member already
checked into an appointment today cannot be checked into a second one — so a
member with two PT sessions the same day can only be marked for one. This
predates the manual button; it just also applies to it.

⚠ **Reversing is not a coach action.** `DELETE /checkins/:id` (front desk,
`checkins.manage`) reverses a visit and returns the session to the balance.
There is no undo on the coach's screen, which is why the confirm-then-disable
flow above matters.
