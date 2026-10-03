# 15 — Sandy messages first (2026-10-03)

> Audience: **mobile app (Flutter)**.

Sandy can now start a conversation with the member at moments that matter:
they stopped coming, their membership is about to end, they missed a PT
session, or their InBody test improved. The texts are written and approved
by the gym.

## Nothing to build for it to work
Each message arrives as:
1. a message from Sandy in a conversation titled **"💚 ساندي"** (the same one is
   reused for later messages) — it's a normal Sandy conversation, so the member
   can reply and Sandy (AI) answers; and
2. an **inbox notification**.

## Build: a mute switch (Settings → Sandy)
| Endpoint | |
| --- | --- |
| `GET /mobile/sandy/nudges` | `{ enabled }` |
| `PATCH /mobile/sandy/nudges` `{ enabled: false }` | mute Sandy messaging me first |

Suggested label: «رسائل من ساندي» — "Sandy can message me with reminders and
encouragement". Default is on.

## Gotchas
- At most one message a day and two a week per member, only during daytime.
- Muting stops only proactive messages — the member can still chat with Sandy.
