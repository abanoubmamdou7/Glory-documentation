# Sandy AI — Medical Documents & X-ray Understanding (update — 2026-09-08)

> **Standalone note for this update.** A dedicated write-up of the new Sandy
> capability shipped today. General Sandy chat lives in
> [09-sandy-ai.md](09-sandy-ai.md); this file covers only the new
> file/X-ray/medical-memory feature.

## What's new in one line

Members can now upload a **medical file** — a lab result (PDF) or an **X-ray /
scan (image)** — and Sandy **reads it, explains it, warns and refers them to a
doctor, and remembers it** on their profile for future chats.

## What Sandy does with a file

1. **Reads it.** PDFs → text extracted server-side. Images (X-rays/scans) → the
   vision model looks at the picture.
2. **Explains it** in plain, friendly Jordanian Arabic — translating the jargon,
   pulling out the concrete facts (lab markers and whether they're in range; or
   the body region and what the image seems to show).
3. **Warns & refers.** It flags anything relevant to safe training and **always
   tells the member to see a doctor**. For an X-ray it says what the image *may*
   show and urges a doctor/ER — it never declares a diagnosis as fact.
4. **Remembers it.** A private summary is stored on the member's profile and fed
   into Sandy's context on every later chat, so it factors known injuries and
   conditions into its advice automatically.

### Two hard guarantees

- **Private to the member + Sandy.** These documents and their summaries are
  **never** shown on the dashboard. There is no staff/coach view of them.
- **Never a diagnosis or treatment.** Sandy explains + warns + refers. It will
  not say "you have a fracture" or recommend medication. Present its reply as
  guidance, not a medical verdict.

---

## API

All endpoints use the **member** bearer token
(`Authorization: Bearer <memberAccessToken>`) and the standard
`{ success, data }` envelope.

| # | Method | Path | Purpose |
| --- | --- | --- | --- |
| 0 | `POST` | `/mobile/uploads` | (existing) Upload the file, get a URL back |
| 1 | `POST` | `/mobile/sandy/documents` | Read + explain + store a medical file |
| 2 | `GET` | `/mobile/sandy/documents` | List my stored medical documents |
| 3 | `GET` | `/mobile/sandy/documents/:id` | Get one stored document |
| 4 | `DELETE` | `/mobile/sandy/documents/:id` | Delete a document (and its memory) |

### The flow is two steps

**Step 1 — upload the file** (existing endpoint, `multipart/form-data`, field
`file`):

```
POST /mobile/uploads
→ { "success": true, "data": { "url": "https://res.cloudinary.com/.../xray.png", ... } }
```

**Step 2 — send that URL to Sandy:**

```json
POST /mobile/sandy/documents
{
  "fileUrl": "https://res.cloudinary.com/.../xray.png",
  "note": "وقعت على إيدي وبتوجعني",   // optional — anything the member wants to add
  "conversationId": "clx…",            // optional — also drop the reply into this chat thread
  "lang": "ar"                          // optional — "ar" (default) | "en"
}
```

**Success `201`:**

```json
{
  "success": true,
  "data": {
    "recordId": "clx…",
    "kind": "IMAGING",
    "title": "صورة أشعة لليد أو الرسغ",
    "reply": "سلامتك. الصورة مش واضحة كفاية… ما بقدر أأكد إذا في كسر. بس بما إنك وقعت وعندك ورم ووجع، لازم تنفحص عند دكتور بأقرب وقت، ووقّف تمارين اليد… إذا لاحظت تشوّه، خدر… توجّه للطوارئ هلأ.",
    "summary": "…",
    "flags": [
      { "severity": "high", "area": "اليد أو الرسغ", "note": "ممكن يدلّوا على كسر أو إصابة — راجع دكتور بأقرب وقت ووقّف تمرين اليد لحد ما تنفحص" }
    ],
    "trainingCaution": true,
    "meta": { "model": "gpt-5.6-luna" }
  }
}
```

| Field | Use |
| --- | --- |
| `reply` | **Render this** as Sandy's chat bubble. |
| `kind` | `LAB_RESULT` · `IMAGING` · `REPORT` · `PRESCRIPTION` · `OTHER`. |
| `title` | Short label for a "My health files" list. |
| `summary` | Plain-language summary (stored). |
| `flags[]` | `{ severity: low\|medium\|high, area?, note }` — surface high-severity ones prominently. |
| `trainingCaution` | `true` = Sandy advised pausing training of some area until a doctor clears it. Good cue for a warning banner. |

If you pass `conversationId`, the upload + Sandy's reply are also written into
that Sandy conversation, so the file shows up in the chat history.

**Errors:** `400` unreadable / oversized (> 15 MB) / unsupported file, or the AI
provider isn't configured · `401` no token / an employee token.

### Managing stored documents

- `GET /mobile/sandy/documents?page=&limit=` → the member's own docs (paginated):
  `[{ id, kind, title, fileUrl, fileType, summary, extracted, flags, trainingCaution, createdAt }]`.
- `GET /mobile/sandy/documents/:id` → one doc. `404` if it isn't the caller's.
- `DELETE /mobile/sandy/documents/:id` → removes the doc **and its memory**.
  `404` if it isn't the caller's.

A "My health files" screen can list these; let the member delete any of them.

---

## Memory recall — how "Sandy remembers" shows up

There is nothing extra to call. Once a document is stored, its summary + flags
are injected into Sandy's per-member context on **every** subsequent
`POST /mobile/sandy/chat`. So a normal question later already accounts for it:

> **Member (new chat, no mention of the injury):** "بدي أبلّش تمرين بنش برس تقيل"
>
> **Sandy:** "بما إن عندك ورم ووجع باليد/الرسغ بعد الوقعة، **لا تبلّشي بنش برس
> تقيل ولا ضغط بالإيدين هلأ**… راجعي دكتور… واحكي مع الكوتش سارة لتعديل برنامجك
> مؤقتًا…"

(Verified live against `gpt-5.6-luna`.)

---

## Gotchas & operational notes

- **⚠ Cloudinary blocks PDF delivery by default.** The upload succeeds, but when
  Sandy tries to fetch the PDF it gets `401` and the request fails. To use PDF
  lab reports, enable **Settings → Security → "Allow delivery of PDF and ZIP
  files"** in the Cloudinary console. **Images (X-rays) work with no change.**
- **Scanned / photo-only PDFs** (no text layer) are rejected with a message to
  upload them as an image instead — the vision path handles those.
- **Max file size: 15 MB.**
- **SSRF-guarded fetch:** Sandy only fetches public URLs; loopback/private hosts
  are refused. Always pass a URL that came from `POST /mobile/uploads`.
- **Not a diagnosis** (repeated because it matters): the reply is safety-forward
  guidance + a doctor referral. Don't relabel it as a medical result in the UI.
- **PHI / privacy:** this stores members' medical data. It's app-private
  (member + Sandy only). For production, consider explicit consent on upload,
  encryption-at-rest for the summaries, and a retention policy. The member can
  delete any record at any time, which also clears it from Sandy's memory.

## Config (backend, FYI)

Reuses Sandy's existing AI provider (`AI_BASE_URL` + `AI_API_KEY` +
`AI_CHAT_MODEL`). The chat model must be **vision-capable** for X-ray/image
understanding (confirmed for `gpt-5.6-luna`). No embedding model needed for this
feature. See [../../docs/API.md](../../docs/API.md) → "Medical documents" for the
full reference.
