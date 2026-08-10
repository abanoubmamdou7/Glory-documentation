# 08 — Coach Chat (member ↔ instructor)

> Audience: **mobile app**. Member Bearer token required on every REST
> endpoint here, and on the Socket.IO connection too. Real-time, **text
> only**, strictly between the member and *their own* assigned instructor.
>
> ⚠ **Scope note:** the member doesn't pick their own instructor — a
> dashboard staff/admin action (`POST /chat/assign-instructor`, see
> [../frontend/10-coach-chat.md](../frontend/10-coach-chat.md)) assigns one,
> which also auto-creates the conversation. There is no "start a new chat"
> button on the member side — if `GET /mobile/chat/conversations` comes back
> empty, the member simply hasn't been assigned an instructor yet.

| # | Method / event | Screen |
| --- | --- | --- |
| 1 | `GET /mobile/chat/conversations` | Chat list (normally just one thread) |
| 2 | `GET /mobile/chat/conversations/:id/messages` | Message history |
| 3 | `PATCH /mobile/chat/conversations/:id/read` | Opening the thread (mark read) |
| 4 | `GET /mobile/chat/unread-count` | Chat tab badge |
| 5 | Socket.IO `message:send` / `message:new` | Sending & receiving live |

---

## 1. `GET /mobile/chat/conversations`

```json
[
  { "id": "...", "memberId": "...", "instructorId": "...",
    "createdAt": "...", "lastMessageAt": "2026-08-10T09:15:00.000Z",
    "instructor": { "id": "...", "fullName": "Coach Ahmed", "avatarUrl": null },
    "unreadCount": 2 }
]
```

Ordered newest-activity-first. In practice this is a **single-item array**
for almost every member — the "current instructor" thread. If a member was
ever reassigned to a different instructor, the old thread stays in this list
(for history) below the new one; don't assume index `0` is always "the"
conversation to open by default without checking `lastMessageAt`/whether the
instructor still matches the member's current one if you surface that
elsewhere in the app.

## 2. `GET /mobile/chat/conversations/:id/messages`

Query: `page`, `limit` (standard pagination). **Page 1 = the most recent
messages** (sorted `createdAt` desc) — reverse the array client-side to
render oldest-at-top/newest-at-bottom, and request `page=2` etc. for
"load older messages" as the user scrolls up.

```json
{ "id": "...", "conversationId": "...", "senderType": "INSTRUCTOR",
  "sender": { "id": "...", "fullName": "Coach Ahmed", "avatarUrl": null },
  "body": "How's the new program going?",
  "read": true, "readAt": "2026-08-10T09:20:00.000Z",
  "createdAt": "2026-08-10T09:15:00.000Z" }
```

`senderType` is `MEMBER` for the member's own messages (bubble on the
right) or `INSTRUCTOR` for the other side (bubble on the left) — don't rely
on comparing `sender.id` to the logged-in member's id for bubble alignment,
`senderType` is the direct, unambiguous signal. `404` if the conversation
isn't this member's own.

## 3. `PATCH /mobile/chat/conversations/:id/read`

Call this when the thread screen opens. Marks every unread message **from
the instructor** as read — `{ "updated": 2 }`. It does not touch the
member's own sent messages (those don't have a "read" concept from the
member's side).

## 4. `GET /mobile/chat/unread-count`

`{ "count": 2 }` — sum across all of the member's conversations, for the
chat tab's badge. Re-fetch on `message:new` (below) rather than polling.

## 5. Sending & receiving — Socket.IO

There is **no REST endpoint to send a message**. Connect once (e.g. at app
launch, alongside the notification-inbox polling) and keep the socket open:

```js
import { io } from 'socket.io-client';

const socket = io(`${BASE_URL}/chat`, {
  auth: { token: memberAccessToken },
});

socket.on('connected', () => { /* ready */ });

socket.on('message:new', (message) => {
  // Append to the open thread if message.conversationId matches, and/or
  // bump the unread badge — this event fires for the member's OWN sent
  // messages too (so every open tab/device stays in sync), not just
  // incoming ones. Check message.senderType to tell which.
});

socket.on('error', ({ message }) => {
  // token invalid/expired -> the socket disconnects right after this;
  // re-authenticate (refresh the access token) and reconnect.
});

function sendMessage(conversationId, body) {
  socket.emit('message:send', { conversationId, body });
}
```

- Reconnect with a **fresh** `accessToken` after a token refresh — the
  socket doesn't pick up a rotated token on its own; disconnect and
  reconnect with the new `auth.token`.
- `body` is required, plain text, max 2000 characters — there is no
  attachment/image support (matches "text only" in the spec this feature
  was built from).
- If the app is backgrounded/offline when a message arrives, it's still
  safely in `ChatMessage` — call `GET .../messages` on resume/foreground to
  catch up rather than relying on having been connected the whole time.

---

## Gotchas checklist

1. No "start new chat" / "pick an instructor" flow exists on the member
   side — assignment is dashboard-driven. An empty conversations list means
   "not assigned yet", not an error state.
2. Sending is **Socket.IO only** — there is deliberately no `POST` message
   endpoint.
3. `message:new` fires for the sender's own messages too (multi-device
   echo) — filter/dedupe by `senderType`/`id` if you're optimistically
   rendering the outgoing bubble before the server ack arrives.
4. Reconnect the socket with a fresh token after any token refresh; a stale
   token gets an `error` event + forced disconnect, not a silent failure.
5. Message history pagination is newest-page-first (`page=1` = latest),
   the opposite of what you might render on screen — reverse client-side.
