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

Today: **30 events — 27 member-facing, 3 staff-facing; 22 immediate, 8
scheduled.**

### Immediate

Membership: `SUBSCRIPTION_STARTED`, `_RENEWED`, `_UPGRADED`, `_FROZEN`,
`_RESUMED`, `_TRANSFERRED`, `_VOIDED`, `PAYMENT_RECEIVED`
· Attendance: `CHECKED_IN`, `SESSION_BOOKED`, `SESSION_CANCELLED`,
`CLASS_CANCELLED`
· Coaching: `COACH_ASSIGNED`, `CHAT_MESSAGE`, `WORKOUT_ASSIGNED`,
`TRAINING_PLAN_READY`, `NUTRITION_PLAN_READY`, `INBODY_RESULT`
· Other: `WELCOME`, `BROADCAST`, and the two staff ones.

### Scheduled (the sweep, every 15 min + on boot)

`SUBSCRIPTION_EXPIRING` (7/3/1 days by default) · `SUBSCRIPTION_EXPIRED` ·
`SESSION_REMINDER` (3h before) · `RATE_YOUR_SESSION` (attended, unrated, 1h–3d
old) · `PAYMENT_DUE` (weekly while overdue) · `INBODY_DUE` (monthly, no test in
30 days) · `SANDY_NUDGE` · `FOLLOW_UP_DUE` *(reserved — not wired yet)*.

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
