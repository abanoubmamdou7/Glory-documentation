# 16 — Admin Assistant (ask the database in plain language)

> Audience: **web dashboard frontend**. A chat panel where an **admin** types a
> question in Arabic or English — *"how many members are active?"*, *"إيراد
> الشهر ده كام؟"*, *"what's member 50001's subscription?"*, *"list the coaches"*
> — and gets a written answer computed from the live database.
>
> Under the hood it turns the question into **one read-only SQL query**, runs it
> safely, and summarises the rows. You don't deal with any of that — you send a
> message and render the answer. But the response also hands you the exact SQL
> and the raw rows, which you can optionally surface for power users.

**This is a different feature from Sandy AI** ([15-sandy-ai.md](15-sandy-ai.md)).
Sandy answers *gym-knowledge* questions for members (training, nutrition) from a
curated knowledge base. This assistant answers questions about *your own
operational data* for admins. Different endpoint, different surface, different
purpose — don't merge them in the UI.

---

## Access & safety (what to tell the user)

- **Gated by the `assistant.use` permission** (2026-09-12; ADMIN bypasses and
  keeps full access). A user **without** `assistant.use` gets **`403`** on every
  route — gate the nav item on that permission (or the ADMIN role). Grant it
  from Team Management like any other permission.
- **Permission-aware — this is new (2026-09-12).** *Which* tables a caller can
  query is scoped by their **other** permissions. An **accountant**
  (`invoices.read` / `payments.manage` / `reports.view`) can ask about billing,
  finance and business metrics but **not** staff/HR data; an **HR** user
  (`staff.read` / `attendance.view.*` / `organization.view`) can ask about staff
  but **not** revenue or salaries. An accountant asking an HR question (or
  vice-versa) is **refused** with `error: "PERMISSION_SCOPE"` and the answer
  names the areas that caller *can* ask about. **ADMIN / grant-all** users reach
  everything readable. See the full map in
  [16a-admin-assistant-permissions.md](16a-admin-assistant-permissions.md).
- **Read-only.** It can never change, add, or delete anything — only read. It
  also cannot see passwords, tokens, OTP codes, or **members' medical records**,
  or other secrets (blocked at the server for **every** caller, ADMIN included).
- It only answers questions about the gym's data. Anything else (general
  knowledge, "delete all members", "show passwords") comes back as a polite
  refusal — see `error` below.

| # | Method | Path | Purpose |
| --- | --- | --- | --- |
| 1 | `POST` | `/admin-assistant/chat` | Ask a question |
| 2 | `GET` | `/admin-assistant/schema` | What tables it can read (optional, for a "what can I ask?" hint) |
| 3 | `GET` | `/admin-assistant/conversations` | List the admin's conversations |
| 4 | `GET` | `/admin-assistant/conversations/:id/messages` | One conversation's history |
| 5 | `PATCH` | `/admin-assistant/conversations/:id` | Rename a conversation |
| 6 | `DELETE` | `/admin-assistant/conversations/:id` | Delete a conversation |

All requests use the normal employee bearer token
(`Authorization: Bearer {{accessToken}}`) and the standard success/error
envelopes described in [../README.md](../README.md).

---

## 1. `POST /admin-assistant/chat` — ask a question

The one endpoint that matters. Send the message; optionally continue an existing
thread by passing its `conversationId`.

**Request body**

```json
{
  "message": "كام عضو حالته active دلوقتي؟",
  "conversationId": "clx...   // optional — omit to start a new conversation"
}
```

- `message` — 1…2000 chars, required. Arabic or English; the answer mirrors the
  language you asked in.
- `conversationId` — optional. Omit for the first message; the response returns
  a new `conversationId` you pass back on follow-ups so the assistant keeps
  context.

**Success `200`**

```json
{
  "success": true,
  "data": {
    "conversationId": "clx1...",
    "messageId": "clx2...",
    "answer": "عدد الأعضاء النشطين دلوقتي هو 5 أعضاء.",
    "sql": "SELECT COUNT(*) AS \"activeMemberCount\" FROM \"Member\" WHERE \"status\" = 'ACTIVE' AND \"deletedAt\" IS NULL LIMIT 500",
    "rowCount": 1,
    "rows": [{ "activeMemberCount": 5 }],
    "error": null,
    "meta": { "model": "gpt-5.6-luna", "latencyMs": 1840 }
  }
}
```

| Field | What to do with it |
| --- | --- |
| `answer` | **The thing you render** — the natural-language reply. Show it as the assistant's chat bubble. |
| `sql` | The exact read-only query that produced the answer, or `null`. Optional to show — nice behind a "Show query" / "details" toggle for power users. |
| `rowCount` | How many rows the query returned (before the 500-row display cap). |
| `rows` | The raw result rows (sensitive columns already stripped), or `null`. Optional — you can render them as a small table under the answer. Capped at 500. |
| `error` | `null` on a normal answer. Otherwise a short code (see below). |
| `meta.model` / `meta.latencyMs` | Observability; optional to show. |

### ⚠ `error` is a product state, not an HTTP error

**Every** chat call that reaches the server returns HTTP **`200`** — including
refusals and failures. Branch on the `error` field, not on the status code:

| `error` | Meaning | Suggested UI |
| --- | --- | --- |
| `null` | Normal answer. | Render `answer` (+ optional `sql`/`rows`). |
| `OUT_OF_SCOPE` | Question isn't about the gym's data (general knowledge, or a destructive request the model refused). | Render `answer` (already a friendly "I can only answer questions about your gym's data"). Don't show it as an error. |
| `PERMISSION_SCOPE` | Question **is** about the gym's data but concerns an area this caller has no permission for (e.g. an accountant asking about staff/HR). | Render `answer` — it already says what they *can* ask about. Don't show it as an error. |
| `UNSAFE_SQL: …` | The model produced a query the guard rejected. | Render `answer` ("couldn't turn that into a safe query, try rephrasing"). |
| `EXEC_ERROR: …` | The query failed to run (e.g. too vague). | Render `answer` ("the query failed, be more specific"). |
| `GENERATION_ERROR` | The AI provider errored while writing the query. | Render `answer` ("something went wrong, try again"). |
| `PROVIDER_NOT_CONFIGURED` | No AI provider is set up on the server yet. | Render `answer`; optionally disable the input until an admin configures it. |

So the render rule is simply: **always show `answer`.** Use `error` only to
decide whether to also show the `sql`/`rows` details (only meaningful when
`error` is `null`).

**Real HTTP errors** (handle these the usual way):
`400` empty/too-long message · `401` no/expired token (refresh & retry — see
[../README.md](../README.md)) · `403` the user lacks `assistant.use` ·
`404` a `conversationId` that isn't this caller's.

### Example questions that work

These all return real answers from the database:

- "كام عضو حالته active؟" · "how many active members do we have?"
- "how many instructors (coaches) are there?"
- "what is the active subscription of member 50001? show package, end date and total"
- "total revenue from paid invoices this month in JOD"
- "list the members whose subscription expires in the next 7 days"
- "which packages are the best selling?"
- "عرض آخر 10 فواتير مدفوعة"

---

## 2. `GET /admin-assistant/schema` — what it can read (optional)

Returns the tables the assistant is allowed to query **for the calling user**
(scoped by their permissions — 2026-09-12) and the blocked ones. Useful if you
want to show a "you can ask about…" hint that matches what *this* user can
actually see. Not needed for the chat to work. An accountant and an HR user get
**different** `tables` lists; ADMIN gets the full set.

```json
{
  "success": true,
  "data": {
    "tables": ["Branch", "Invoice", "Member", "Package", "Payment", "Subscription", "Transaction", "..."],
    "blockedTables": ["RefreshToken", "MemberRefreshToken", "MemberOtp", "GymCheckInToken", "MemberMedicalRecord", "_prisma_migrations"],
    "blockedColumns": ["password", "embedding", "searchvector", "codehash", "tokenhash", "secret", "refreshtoken"]
  }
}
```

---

## 3. `GET /admin-assistant/conversations` — list threads

Paginated (`?page=&limit=`), newest activity first. Scoped to the calling admin.

```json
{
  "success": true,
  "data": [
    { "id": "clx1...", "title": "كام عضو نشط", "createdAt": "2026-09-02T09:00:00.000Z", "lastMessageAt": "2026-09-02T09:03:00.000Z", "messageCount": 6 }
  ],
  "meta": { "page": 1, "limit": 20, "total": 3, "totalPages": 1 }
}
```

`title` is auto-derived from the first message (first 60 chars); let the user
rename it (endpoint 5). Use this to build a left-hand conversation sidebar, same
as any chat UI.

---

## 4. `GET /admin-assistant/conversations/:id/messages` — history

Paginated. **Page 1 is the most recent page**; messages **within** a page are
ordered oldest→newest, so you can render a page directly top-to-bottom.

```json
{
  "success": true,
  "data": [
    { "id": "m1", "sender": "USER", "body": "كام عضو نشط؟", "sql": null, "rowCount": null, "error": null, "model": null, "latencyMs": null, "createdAt": "..." },
    { "id": "m2", "sender": "ASSISTANT", "body": "عدد الأعضاء النشطين 5.", "sql": "SELECT COUNT(*) ...", "rowCount": 1, "error": null, "model": "gpt-5.6-luna", "latencyMs": 1840, "createdAt": "..." }
  ],
  "meta": { "page": 1, "limit": 20, "total": 6, "totalPages": 1 }
}
```

- `sender` is `USER` or `ASSISTANT`.
- Assistant turns carry the **audit trail**: the `sql` that ran, `rowCount`,
  `error`, `model`, `latencyMs`. (Note: the raw `rows` are **not** persisted —
  only the live `POST /chat` response includes `rows`. History keeps the answer
  text + the SQL, which is what matters for audit.)
- `404` if the conversation isn't this admin's.

---

## 5. `PATCH /admin-assistant/conversations/:id` — rename

```json
{ "title": "تقارير الإيراد" }
```

Body `title` 1…120 chars. `404` if not the caller's conversation.

## 6. `DELETE /admin-assistant/conversations/:id` — delete

Deletes the conversation and all its messages. Returns
`{ "id": "...", "deleted": true }`. `404` if not the caller's.

---

## Building the chat UI

It's a standard chat panel — the same shape as Coach Chat or Sandy, minus any
real-time layer (this is plain request/response, no Socket.IO):

1. **Sidebar** — `GET /conversations`; a "New chat" button just clears the
   current `conversationId` in your state.
2. **Thread** — when a conversation is selected, `GET …/:id/messages` and render
   the bubbles.
3. **Send** — `POST /chat` with `{ message, conversationId }`. On the first
   message omit `conversationId` and **save the one from the response**. Append
   the user bubble optimistically; append the assistant bubble from `answer`.
4. **Details (optional)** — under an assistant bubble, a collapsible "Show query
   / data" that renders `sql` (monospace) and `rows` (a small table). Only when
   `error === null`.
5. **Loading** — answers take a couple of seconds (the server makes two model
   calls: write-SQL then summarise). Show a typing indicator; the request is a
   single awaitable call.
6. **Empty state** — offer a few example chips (see the list in §1). Optionally
   pull `GET /schema` to hint what's answerable.

### Gotchas

- **Don't treat `error` as a failure toast.** It's a normal `200`; the `answer`
  is already user-friendly. Only `400/401/403/404` are real errors.
- **RTL** — answers are frequently Arabic; render assistant bubbles with
  `dir="auto"`.
- **Money** is JOD; the answer already formats amounts, so just print `answer`.
  If you render `rows` yourself, money columns are decimal strings.
- **Latency** — a few seconds per answer is normal (two model calls). Disable
  the send button while awaiting.
- **Scope** — gate the nav item on the `assistant.use` permission (ADMIN always
  has it). A user without it gets `403` on every route. And remember the
  per-caller table scoping: a scoped user's out-of-area question comes back as a
  `200` with `error: "PERMISSION_SCOPE"`, not a `403` — see
  [16a-admin-assistant-permissions.md](16a-admin-assistant-permissions.md).
- **Availability** — if `error` is `PROVIDER_NOT_CONFIGURED`, the server has no
  AI credentials yet; the feature is effectively off until an admin sets them.

---

## Configuration (backend, FYI)

The assistant needs an AI provider configured on the server (`AI_BASE_URL`,
`AI_API_KEY`, `AI_CHAT_MODEL` — any OpenAI-compatible endpoint). No embedding
model is required (this is text-to-SQL, not the RAG that Sandy uses). Until the
key is set, chat returns `200` with `error: "PROVIDER_NOT_CONFIGURED"`.

See [../../docs/API.md](../../docs/API.md) → "Admin Assistant" for the full
reference and the exact SQL-guard rules.
