# 10 — Coach Chat (member ↔ instructor)

> Audience: **web dashboard frontend** — two different surfaces:
> 1. The **admin dashboard**'s "assign an instructor to a member" action
>    (wherever that lives in your UI — a member's profile, a subscription
>    screen, etc.), gated by `chat.manage`.
> 2. The **instructor's own dashboard** (the separate surface from
>    [09-instructor-dashboard.md](09-instructor-dashboard.md)) — a "Coach
>    Chat" nav item showing the instructor's conversations with their
>    assigned members.
>
> Real-time delivery for both is **Socket.IO**, **text only**.

⚠ **Read this before wiring anything up:** the "natural" trigger for this
feature is a member subscribing and getting a trainer assigned
(`Subscription.trainerId`) — but **Phase 6 (Sales/Subscriptions lifecycle)
doesn't exist yet**, so nothing in the codebase ever sets that field today.
Rather than block this feature on building all of Sales/Subscriptions, a
small dedicated field (`Member.assignedInstructorId`) + one endpoint
(`POST /chat/assign-instructor`) serve as the **interim** trigger: calling
it sets the assignment **and** auto-creates the chat conversation in the
same call. Wire your "assign instructor" UI action to this endpoint now;
when Phase 6 ships, the subscription-creation flow is expected to call the
same endpoint (or the trigger moves server-side) — no client changes
anticipated either way.

| # | Method / event | Surface |
| --- | --- | --- |
| 1 | `POST /chat/assign-instructor` | Admin: "assign instructor" action |
| 2 | `GET /coach-chat/conversations` | Instructor dashboard: chat list |
| 3 | `GET /coach-chat/conversations/:id/messages` | Instructor dashboard: message history |
| 4 | `PATCH /coach-chat/conversations/:id/read` | Instructor dashboard: opening a thread |
| 5 | `GET /coach-chat/unread-count` | Instructor dashboard: chat nav badge |
| 6 | Socket.IO `message:send` / `message:new` | Instructor dashboard: sending & receiving live |

---

## 1. `POST /chat/assign-instructor` — admin dashboard

**Permission:** `chat.manage` (ADMIN bypasses).

```json
{ "memberId": "...", "instructorId": "..." }
```

**201:** the `ChatConversation` row. Safe to call repeatedly for the same
pair (idempotent, returns the same conversation — never duplicates).
Calling it with a **different** `instructorId` for a member who already has
one **reassigns** them — their old conversation with the previous
instructor is kept (not deleted), a new one is created for the new pairing.

**Errors:** `400` unknown `memberId`, or `instructorId` that isn't a real
`User` with `role=INSTRUCTOR`; `403` missing `chat.manage`.

## 2. `GET /coach-chat/conversations` — instructor dashboard

**Role-restricted to `INSTRUCTOR`** — this whole `/coach-chat/*` surface is
personal data, like `/mobile/profile` on the member side; a `STAFF`/`ADMIN`
token gets `403`. There's no oversight/monitoring view for other roles in
this design.

```json
[
  { "id": "...", "memberId": "...", "instructorId": "...",
    "createdAt": "...", "lastMessageAt": "2026-08-10T09:15:00.000Z",
    "member": { "id": "...", "fullName": "Ahmed Hossam", "avatarUrl": null },
    "unreadCount": 1 }
]
```

Ordered newest-activity-first — render as the chat sidebar's conversation
list, badge = `unreadCount` per row.

## 3. `GET /coach-chat/conversations/:id/messages` — instructor dashboard

Standard pagination; **page 1 = most recent messages** (reverse
client-side for a top-old/bottom-new thread view).

```json
{ "id": "...", "conversationId": "...", "senderType": "MEMBER",
  "sender": { "id": "...", "fullName": "Ahmed Hossam", "avatarUrl": null },
  "body": "Can we move today's session?",
  "read": false, "readAt": null, "createdAt": "..." }
```

`senderType` (`MEMBER`/`INSTRUCTOR`) drives bubble alignment directly —
don't compare `sender.id` to the logged-in instructor's id. `404` if the
conversation isn't this instructor's own.

## 4. `PATCH /coach-chat/conversations/:id/read` — instructor dashboard

Call on opening a thread. Marks the member's unread messages as read —
`{ "updated": 1 }`.

## 5. `GET /coach-chat/unread-count` — instructor dashboard

`{ "count": 1 }` — across all of the instructor's conversations, for the
"Coach Chat" nav badge.

## 6. Sending & receiving — Socket.IO

No REST send endpoint exists — send exclusively over the socket:

```js
import { io } from 'socket.io-client';

const socket = io(`${BASE_URL}/chat`, {
  auth: { token: instructorAccessToken }, // the instructor's own employee access token
});

socket.on('connected', () => { /* ready */ });

socket.on('message:new', (message) => {
  // append to the matching open thread + refresh the conversation list's
  // lastMessageAt/unreadCount. Fires for the instructor's own sent
  // messages too (multi-tab sync) — check senderType.
});

socket.on('error', ({ message }) => {
  // bad/expired token, or a non-INSTRUCTOR employee token -> forced
  // disconnect right after this fires.
});

socket.emit('message:send', { conversationId, body: 'Sure, how about 5pm?' });
```

Reconnect with a fresh token after any refresh — the socket does not
re-authenticate itself on token rotation.

---

## Gotchas checklist

1. `/coach-chat/*` is `INSTRUCTOR`-only — testing it with a `STAFF`/`ADMIN`
   token (even one with `chat.manage`) gets `403`; `chat.manage` only gates
   the *assign* action on the admin side.
2. There is no admin/staff view into a specific chat's messages — by design,
   this stays strictly member ↔ their own instructor.
3. `POST /chat/assign-instructor` is the **only** way a conversation gets
   created — there's no separate "start chat" action to wire up.
4. Sending is Socket.IO-only on both surfaces; don't look for a `POST
   .../messages` REST endpoint, it doesn't exist.
5. This is the interim trigger described in the callout above — if/when
   Phase 6 (Sales/Subscriptions) ships with its own trainer-assignment UI,
   check back here before assuming this endpoint is still the right one to
   call.
