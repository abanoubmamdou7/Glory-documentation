# Notifications & device push — what fires, who gets it, and how to wire a client

*Added 2026-10-03, when real FCM delivery went live. Before this date every
"notification" was an in-app inbox row and the delivery counters were
approximations.*

---

## 1. The one thing to understand first

Every notification is written to the **in-app inbox first**, then pushed to the
member's devices.

```
event happens → PushNotification + MemberNotification row  (durable)
              → FCM push to that member's devices            (best effort)
```

So: a phone that is off, uninstalled, or has notifications denied **never
loses the message** — it is waiting in `GET /mobile/notifications` on next
open. A Firebase outage costs a buzz, not data.

Two consequences for the UI:

- The inbox is the source of truth. Build the bell badge off
  `GET /mobile/notifications/unread-count`, never off push receipts.
- `pushEnabled: false` on a member silences the **push**, not the inbox row.
  The toggle means "don't buzz my phone", not "hide my history".

---

## 2. Mobile client checklist

| When | Call |
| --- | --- |
| After login | `POST /mobile/devices` with the FCM token |
| On FCM token refresh (`onTokenRefresh`) | `POST /mobile/devices` again — idempotent |
| On logout | `DELETE /mobile/devices` with that token |

**Registering is idempotent on the token**, so calling it on every app start is
fine and is in fact what you want: FCM hands the *same* token back after a
reinstall, so re-registering re-owns the row and revives a token the server had
retired as dead.

⚠ **`DELETE` on logout is not optional.** Skip it and a shared or sold handset
keeps receiving the previous member's notifications until FCM happens to
invalidate the token.

### The payload you receive

```jsonc
{
  "notification": { "title": "اشتراكك صار فعّال ✅", "body": "..." },
  "data": {
    "event": "SUBSCRIPTION_STARTED",
    "screen": "subscriptions",        // where to navigate on tap
    "notificationId": "cku…",          // mark-read via the inbox endpoint
    "subscriptionId": "cku…"           // whichever ids the event carries
  }
}
```

Deep-link on `data.screen`. Treat an unknown `screen` as "open the inbox"
rather than crashing — new events get added.

---

## 3. Coach / staff app

Employees can register a device too (`POST /devices`, no permission needed) and
receive:

- `STAFF_MEMBER_MESSAGE` — one of their members sent a chat message
- `STAFF_NUTRITION_PLAN_PENDING` — Sandy drafted a plan awaiting their review
- `SESSION_CANCELLED` — a reservation of theirs was cancelled

⚠ **Employees have no inbox.** There is no `GET /staff/notifications` — a staff
alert reaches the handset or nowhere. If the coach app needs a durable list,
that is a new feature, not a missing endpoint.

---

## 4. "Do we notify about X?" — ask the API

```
GET /notifications/events
```

Returns one row per event with `when` it fires, `audience`, the deep-link
`screen`, and the `dedupe` key for scheduled ones. It also returns
**`undocumented`** — events the code knows about that the catalog does not
describe. **That array should always be empty**; if it isn't, you have found a
real gap.

Today: **50 events — 34 member-facing, 16 employee-facing; 32 immediate, 18
scheduled.**

### What the member can switch off (new, 2026-10-15)

`GET /mobile/notifications/preferences` → `{ pushEnabled, events: [{ event, muted }] }`
and `PATCH` the same path with `{ muted: [...] }` to **replace** the set (`[]`
turns everything on). Build the settings screen straight off `events` — it is
generated from the server's own catalog, so a new notification type appears in
the list without a client release.

Three things to get right:
1. **A muted type still appears in the inbox.** Muting stops the *push*, not the
   record — same meaning as the old all-or-nothing `pushEnabled`. Don't hide
   muted types from the inbox list.
2. **`BROADCAST` is not in the list and cannot be muted** — a gym-wide
   announcement has to arrive. Sandy nudges are also absent: they have their own
   switch at `PATCH /mobile/sandy/nudges`.
3. **`pushEnabled` wins.** With the master switch off, nothing buzzes no matter
   what these say — show it above the per-type list, not beside it.

### Quiet hours

Gym-wide, set by the dashboard at `PATCH /settings/notifications`
(`quietHoursEnabled`, `quietHoursStart`, `quietHoursEnd` — Amman hours,
default **22:00 → 08:00**, window wraps midnight). They apply to **scheduled**
notifications only; transactional ones still arrive, because they answer
something the member just did. A reminder caught by the window is **not lost** —
it is simply sent once the window opens.

### The coach's bell

Staff notifications live in `GET /employee-notifications` (+ `PATCH :id/read`,
`PATCH read-all`). They are written there **before** any push is attempted, so
the bell is correct even for an employee who never registered a device. The
per-employee switches on `PATCH /settings/notifications/me` now genuinely gate
the push (`memberMessages`, `ptSessions`, `classBookings`,
`followUpReminders`); target decisions, contract expiries and the daily brief
have no switch and always arrive.

### Immediate

Membership: `SUBSCRIPTION_STARTED`, `_RENEWED`, `_UPGRADED`, `_FROZEN`,
`_RESUMED`, `_TRANSFERRED`, `_VOIDED`, `PAYMENT_RECEIVED`
· Attendance: `CHECKED_IN`, `SESSION_BOOKED`, `SESSION_CANCELLED`,
`CLASS_CANCELLED`
· Coaching: `COACH_ASSIGNED`, `CHAT_MESSAGE`, `WORKOUT_ASSIGNED`,
`WORKOUT_STARTED`, `WORKOUT_COMPLETED`, `WORKOUT_CANCELLED`,
`TRAINING_PLAN_READY`, `NUTRITION_PLAN_READY`, `INBODY_RESULT`
· Other: `WELCOME`, `BROADCAST`, and the four employee ones.

**For the coach's screens:** `STAFF_MEMBER_ASSIGNED` lands in the employee
inbox (`GET /employee-notifications`) whenever a member becomes that coach's —
from the dashboard's assign button *or* from a PT sale that auto-assigns.
`data.memberId` deep-links to the member profile. The **previous** coach gets
a separate "moved to another coach" notification, so a reassignment produces
two rows for two different people; a reassignment to the same coach produces
none.

### Scheduled (the sweep, every 15 min + on boot)

`SUBSCRIPTION_EXPIRING` (**15/7/3/1 days** by default — 15 and 7 are the
gym's asks, 3 and 1 are last-chance nudges; set `REMINDER_EXPIRY_DAYS` to
change, e.g. `15,7` for only the two) · `SUBSCRIPTION_EXPIRED` ·
`SESSION_REMINDER` (3h before) · `RATE_YOUR_SESSION` (attended, unrated, 1h–3d
old) · `PAYMENT_DUE` (weekly while overdue) · `INBODY_DUE` (**the 1st of every
month**, Amman time, from 10:00 — every member with a live membership is told
to come in for an InBody; anyone who tested in the last 14 days is skipped
until the next month. Tune with `REMINDER_INBODY_HOUR` /
`REMINDER_INBODY_SKIP_IF_TESTED_DAYS`) · `SANDY_NUDGE` · `FOLLOW_UP_DUE` *(reserved — not wired yet)*.

**Every scheduled send is deduplicated by a unique key**, so the sweep running
96 times a day cannot spam anyone. `POST /notifications/reminders/run` with
`{"dryRun": true}` shows what would go out.

---

## 5. Diagnosing "members say they get no notifications"

`GET /devices/status` (needs `notifications.manage` or `settings.view`):

```jsonc
{
  "firebase": { "configured": true, "projectId": "…", "error": null },
  "liveTokens": { "members": 412, "employees": 9 },
  "retiredTokens": 37,
  "byPlatform": { "ANDROID": 300, "IOS": 112, "WEB": 0 },
  "reach": { "activeMembers": 980, "membersWithADevice": 412, "coveragePct": 42.0 }
}
```

Read it in this order:

1. **`firebase.configured: false`** → push was never set up on this
   environment. Everything is inbox-only. Not a bug, a missing credential.
2. **`reach.coveragePct`** → the ceiling on any broadcast. 42 % means a
   "send to all members" can reach at most 42 % of them on their phones, no
   matter what the notification says. Usually the app hasn't shipped the
   device-registration call, or members never granted permission.
3. **`noTokenCount` on the notification row** → who was targeted but had no
   live device.
4. **`retiredTokens`** → tokens FCM declared dead and we retired. Normal and
   healthy; it climbs as people change phones.

---

## 6. Gotchas

1. **Counters are per-device, not per-member.** A member with a phone and a
   tablet contributes 2 to `successCount` and 1 to `totalCount`. `totalCount`
   is people; `successCount`/`failedCount` are deliveries.
2. **Amounts, chat text and medical values are never in a notification** —
   they'd show on a lock screen. "Payment received", not "150 JOD received".
   If the UI needs the detail, read it from the deep-linked screen.
3. **Arabic is the default on the wire.** The push title/body use the Arabic
   copy when present (the gym is Jordanian); the inbox row carries both, so
   render by the member's `appLanguage` there.
4. **A sale sends one notification, not two.** A payment taken inside
   `POST /sales/checkout` does *not* fire `PAYMENT_RECEIVED` — that member
   already gets `SUBSCRIPTION_STARTED`. Only `POST /payments` notifies.
5. **A PT sale's coach assignment doesn't fire `COACH_ASSIGNED`.** That event
   is for a deliberate desk assignment (`POST /chat/assign-instructor`); a
   coach attached automatically by a sale is covered by the sale's own
   notification. (The reason is correctness: the sale's assignment runs inside
   the sale transaction, and notifying from there would tell a member they have
   a coach even when the sale is then rolled back.)
6. **Don't poll the sweep.** It runs itself; the endpoint is for a catch-up or
   a `dryRun`.
