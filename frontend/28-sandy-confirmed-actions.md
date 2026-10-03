# Sandy executes only a confirmed action — dashboard guide (2026-10-03)

Sandy can now *prepare* a real operation (freeze a subscription, renew, assign
a coach, record a payment…) from the chat or from an advisor recommendation.
**She never runs it.** The dashboard shows a confirm button; only that button
— `POST /admin-assistant/actions/:id/execute` with `confirm: true` — writes,
and the server re-checks the user's permission at that moment.

Full reference: `docs/API.md` → "Sandy executes only a confirmed action".
Postman: folder **"Admin Assistant — Sandy Actions (confirm to execute)"**.

## 1. What changed in existing responses

| Where | Before | Now |
|---|---|---|
| `POST /admin-assistant/chat` | `{ …, error }` | adds `pendingAction` (object or `null`), and `candidates` / `permission` / `missing` on some errors |
| `GET /admin-assistant/recommendations` and the recommendations inside any insights report | `action` = the text step | ⚠ **`action` = the executable action object or `null`**. The text step is now `summary` (also `actionText`). New: `permission`, `extraPermission` |
| `?status=` on recommendations | `PROPOSED / ACCEPTED / DISMISSED / DONE` | also `OPEN` (= `PROPOSED`) |
| `PATCH /admin-assistant/recommendations/:id` | note optional | `DISMISSED` **requires** `note` ≥ 3 chars (`400 REASON_REQUIRED`) |
| chat history (`GET …/conversations/:id/messages`) | | each message has `actionId` (the action it proposed, or `null`) |

Update any code that renders `recommendation.action` as text — use `summary`.

## 2. The chat

`POST /admin-assistant/chat` stays one request and never writes. Render by `error`:

| `error` | Show |
|---|---|
| `null` + `pendingAction` | the answer, then an **action card**: `pendingAction.summary`, a primary button labelled `pendingAction.confirmText`, a "Preview" link, a "Cancel" link, and the expiry (30 min) |
| `null`, no `pendingAction` | a normal read answer |
| `ACTION_AMBIGUOUS` | the answer + a picker from `candidates` (members: `memberCode`, `fullName`; sales: `packageName`, `membershipType`, `status`, `endDate`). Picking one = send the request again with the code |
| `ACTION_INCOMPLETE` | the answer (it asks for what is missing; `missing` lists the fields) |
| `ACTION_NOT_FOUND` | the answer |
| `PERMISSION_DENIED` | the answer + "needs `{permission}`" |
| `WRITE_REFUSED` | the answer (deletes, price changes and permission changes are never available) |

If the user types «نفّذ» / "yes", the chat answers that execution is the
button — it does not execute. Don't send chat text as a confirmation.

On reopening a conversation, show confirm buttons only for
`GET /admin-assistant/actions/pending?conversationId=…` (still PENDING, not
expired).

## 3. The decision board

For each recommendation:

- `action === null` → read-only card (Accept / Dismiss / Done as before).
- `action` present and `action.status === 'PENDING'` → add the primary
  **Execute** button labelled `action.confirmText`. Show it only if the user
  holds `action.permission` (and `action.extraPermission` if set) — check
  `GET /auth/me` permissions; ADMIN has all.
- **Execute** = `POST /admin-assistant/actions/{action.id}/execute`. On
  success the recommendation becomes `DONE` (`decision` in the response).
- **Accept** (`PATCH status ACCEPTED`) only records the decision; it does not
  execute. Don't wire Accept to execute.
- **Dismiss** needs a reason field (≥ 3 chars); it cancels the attached action.

## 4. The confirm dialog

1. `POST /admin-assistant/actions/:id/preview` (body `{}`) → render `effects`
   (e.g. "end date 2026-10-23 → 2026-11-06", "fee 5.000 JOD",
   "balance 85.500 → 75.500"). If `wouldBeRefused` is set, show its message and
   disable the button (or, for `FEE_ALREADY_PAID`, explain a refund is needed).
2. Execute with `{ confirm: true, note? }` and an `Idempotency-Key` header —
   generate one UUID when the dialog opens and reuse it on retries, so a
   double-click or a network retry never runs twice.
3. Show `result` / success, refresh the related screen.

Send only `confirm` and `note`. Sending `target`, `body` or `key` is a `400` —
the server's stored action is the only source of the route and body.

## 5. Errors

Envelope: `{ success: false, statusCode, error, message, ...extra }`. Show
`message`; branch on `error`:

| `error` | UI |
|---|---|
| `CONFIRM_REQUIRED`, `ACTION_BODY_NOT_ALLOWED` | a bug in the dashboard — log it |
| `ACTION_INVALID` | "this action can no longer run" (+ `fields`) |
| `PERMISSION_DENIED` | "needs `{permission}`" |
| `ALREADY_EXECUTED` | "already done by someone else" → refresh |
| `ACTION_CANCELLED`, `ACTION_IN_PROGRESS`, `ACTION_FAILED` | refresh the card |
| `FEE_ALREADY_PAID` | "a fee of `{amount} {currency}` was collected"; offer `replacementAction` (same card, now with the refund) — the original is cancelled |
| `ACTION_EXPIRED` (410) | "expired — ask Sandy again" |
| `ACTION_REJECTED` (422) | the route's own reason (e.g. "Only an ACTIVE subscription can be frozen"); the card stays PENDING |

## 6. Audit screen (optional)

`GET /admin-assistant/actions/audit?page=&limit=&key=&result=&event=` (needs
`assistant.insights`): who proposed, previewed, executed, cancelled or was
refused, with the route and error code. Payment rows show only amount + method.
