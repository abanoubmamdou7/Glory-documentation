# Notification settings (mobile)

**New 2026-10-15.** The member can now switch off individual kinds of
notification, not just all of them. This is the screen behind
"الإشعارات" in settings.

Two endpoints, member bearer token:

| | |
| --- | --- |
| `GET /mobile/notifications/preferences` | what can be silenced, and what is |
| `PATCH /mobile/notifications/preferences` | replace the silenced set |

## The response

```json
{
  "pushEnabled": true,
  "events": [
    { "event": "BIRTHDAY", "muted": false },
    { "event": "INBODY_DUE", "muted": true },
    { "event": "SUBSCRIPTION_EXPIRING", "muted": false }
  ]
}
```

**Render the list straight off `events`.** It is generated from the server's own
notification catalog, so when the gym adds a new kind of notification it shows
up in this screen with no app release. Label each row from your own
translations keyed by `event` — and fall back to the raw key rather than hiding
a row you don't have a label for yet, otherwise a new type becomes invisible and
un-mutable.

## Saving

`PATCH` with the **complete** list you want silenced:

```json
{ "muted": ["BIRTHDAY", "INBODY_DUE"] }
```

It replaces the set, so `{ "muted": [] }` turns everything back on. The response
is the same shape as the `GET`, so use it to re-render without a second call.

`400` if you send an employee-only event, `BROADCAST`, or an unknown value.

## Three things to get right

1. **A muted notification still appears in the inbox.** Muting stops the push —
   the phone doesn't buzz — but the record is still written, exactly like the
   older `pushEnabled` switch. Don't filter muted types out of
   `GET /mobile/notifications`; the member turned off the interruption, not
   their own history.
2. **`pushEnabled` is the master switch and it wins.** With it off nothing
   buzzes regardless of these per-type settings, so put it at the top of the
   screen as a single toggle and grey out (or clearly subordinate) the list
   below it. It lives on `PATCH /mobile/profile/notifications`, not here.
3. **Two things are deliberately absent from the list.** `BROADCAST` cannot be
   muted — a gym-wide announcement ("the branch is closed tomorrow") has to
   arrive. Sandy's proactive nudges aren't here either; they have their own
   switch at `PATCH /mobile/sandy/nudges` (see
   [15-sandy-proactive-messages.md](15-sandy-proactive-messages.md)). Don't
   build a second control for them.

## Quiet hours are not a member setting

The gym sets one quiet-hours window for everyone from the dashboard (default
**22:00–08:00** Amman). It holds *scheduled* notifications — reminders, digests
— until morning; transactional ones (payment received, checked in) still arrive,
because they answer something the member just did. Nothing is lost: a reminder
caught by the window goes out when it opens.

There is no per-member quiet-hours setting, so don't show one.

## What a member might now receive

Beyond the existing membership/payment/booking notifications, 2026-10-15 added:
a **birthday** greeting, a **follow-up due** nudge on the day it's expected, a
**promo expiring** warning (only for a promo they actually used), and — for a
member who is a **guardian** — their dependent's membership expiring and their
dependent checking in. All of them are mutable.
