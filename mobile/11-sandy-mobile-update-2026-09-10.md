# Sandy AI — Mobile Integration Update (for the mobile developer)

**Date:** 2026-09-10 · **Backend:** Glory Gym API · **Scope:** the member Sandy chat

Two updates. **Nothing breaks** — your current `POST /mobile/sandy/chat`
integration keeps working unchanged. One item needs (almost) nothing from you;
the other is optional but recommended (live typing).

---

## ✅ TL;DR — what you need to do

1. **Message formatting** → basically nothing. Just render the `answer` string
   **verbatim** with `dir="auto"` (RTL). It's now clean, organized plain text
   (emoji headers, bullets) — the old `###` / `**` garbage is gone.
2. **Live typing (optional)** → switch the send button to the new streaming
   endpoint `POST /mobile/sandy/chat/stream` and render tokens as they arrive.
   Sample Flutter code below.

---

## 1) Message formatting — the `###` / `**` problem is fixed

**What you saw:** Sandy's answers showed literal `###` and `**الفطور**` because
the app renders the text as-is and Sandy was sending Markdown.

**What changed (backend):** Sandy now writes **clean plain text** — emoji
section headers (`🍳 الفطور`, `🍗 الغدا`, `💪 التمرين`, `📈 المتابعة`), `- `
bullets, blank lines between sections. It no longer emits `#`/`##`/`###`
headings or `**bold**`.

**What you do:**
- Render `answer` **as-is**, `dir="auto"`, preserve line breaks
  (`\n` → new lines; e.g. `Text(answer)` with default soft-wrap, or a
  `SelectableText`). That's it — it looks organized now.
- **Optional / nicer:** add a Markdown renderer (`flutter_markdown`) so lists
  and any future emphasis render richly. **Not required** — the emoji headers +
  `- ` bullets are valid Markdown too, so it renders fine either way.

There is **no API change** for this — same `answer` field on the same endpoints.

---

## 2) Live typing — new streaming endpoint (optional)

For a ChatGPT-style typing effect, use the new SSE endpoint instead of `/chat`.

**Request** — identical to `/chat`:
```
POST /mobile/sandy/chat/stream
Authorization: Bearer <memberAccessToken>
Content-Type: application/json
Accept: text/event-stream

{ "message": "قديش بروتين بحتاج باليوم؟", "conversationId": "clx… (optional)" }
```

**Response:** `Content-Type: text/event-stream` — a sequence of frames, each is
an `event:` line + a `data:` line (JSON), separated by a blank line:

| `event:` | `data` payload | What to do |
| --- | --- | --- |
| `delta` | `{ "text": "…" }` | Append `text` to the current bubble. |
| `done`  | `{ conversationId, messageId, citations, refusalReason, meta }` | Stream finished + saved. Now show citation chips + a refusal badge if `refusalReason != null`. |
| `end`   | `{}` | Always last — close the stream. |
| `error` | `{ message, statusCode }` | Stream couldn't start (e.g. a `conversationId` that isn't this member's). No `delta`/`done` will come. |

**Rules:**
- `citations` and `refusalReason` are **only in `done`** — not on deltas. Stream
  the text first, decorate on `done`.
- It's **always HTTP 200** once the stream starts. Don't rely on status codes —
  branch on the frame `event` (`error` for the foreign-conversation case).
- The **first `delta` can take a few seconds** (the model reasons before
  emitting). Show a "typing…" indicator until the first delta arrives.
- A refusal (off-topic / provider down) arrives as a **single** `delta` then
  `done` with a non-null `refusalReason` — render it like any normal answer.
- The message is **saved server-side** exactly like `/chat`, so it shows up in
  `GET /mobile/sandy/conversations/:id/messages` afterward.
- `401` if the token is missing/expired or it's an employee token (the guard
  runs before the stream opens).

### Flutter sample (using the `http` package)

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

// Returns the final "done" payload; calls onDelta for each streamed chunk.
Future<Map<String, dynamic>?> askSandyStream({
  required String baseUrl,
  required String memberToken,
  required String message,
  String? conversationId,
  required void Function(String text) onDelta,
}) async {
  final req = http.Request('POST', Uri.parse('$baseUrl/mobile/sandy/chat/stream'))
    ..headers['Authorization'] = 'Bearer $memberToken'
    ..headers['Content-Type'] = 'application/json'
    ..headers['Accept'] = 'text/event-stream'
    ..body = jsonEncode({
      'message': message,
      if (conversationId != null) 'conversationId': conversationId,
    });

  final res = await http.Client().send(req);              // streamed response
  final lines = res.stream.transform(utf8.decoder);

  String buffer = '';
  Map<String, dynamic>? done;

  await for (final chunk in lines) {
    buffer += chunk;
    int sep;
    while ((sep = buffer.indexOf('\n\n')) != -1) {
      final frame = buffer.substring(0, sep);
      buffer = buffer.substring(sep + 2);

      String event = 'message';
      String data = '';
      for (final line in frame.split('\n')) {
        if (line.startsWith('event:')) event = line.substring(6).trim();
        if (line.startsWith('data:')) data = line.substring(5).trim();
      }
      if (data.isEmpty) continue;
      final payload = jsonDecode(data);

      switch (event) {
        case 'delta':
          onDelta(payload['text'] as String);            // append to the bubble
          break;
        case 'done':
          done = payload as Map<String, dynamic>;         // citations, refusalReason, ids, meta
          break;
        case 'error':
          throw Exception(payload['message'] ?? 'Sandy failed');
        case 'end':
          break;                                          // stream finished
      }
    }
  }
  return done;
}
```

Usage:
```dart
final done = await askSandyStream(
  baseUrl: baseUrl,
  memberToken: token,
  message: text,
  conversationId: currentConversationId,   // null on the first message
  onDelta: (t) => setState(() => bubbleText += t),
);
// after the stream:
currentConversationId = done?['conversationId'];
final citations = done?['citations'] as List? ?? [];
final refusal   = done?['refusalReason'];         // null on a normal answer
```

> Keep using `POST /mobile/sandy/chat` (single JSON response) wherever you don't
> want streaming — both endpoints coexist.

---

## What did NOT change

- `POST /mobile/sandy/chat` — same request/response, same `answer`/`citations`/
  `refusalReason`/`meta` fields.
- Conversations, history, suggestions, and the medical-documents endpoints —
  all unchanged.
- Auth, tokens, envelopes — unchanged.

Questions? Full reference: [09-sandy-ai.md](09-sandy-ai.md) and
[../../docs/API.md](../../docs/API.md) (Sandy AI section).
