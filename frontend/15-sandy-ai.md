# Sandy AI — knowledge base & oversight (dashboard)

Sandy is the gym's AI assistant. Members talk to it from the mobile app
([mobile/09-sandy-ai.md](../mobile/09-sandy-ai.md)); the **dashboard** owns the
knowledge base that its answers come from, plus read-only oversight of what
members have been asking.

Base path `/sandy-ai`, employee bearer token. Reads need `sandy.read`, knowledge
writes need `sandy.manage` — **ADMIN bypasses both** (new permission group
"Sandy AI", catalog now 36 keys).

---

## 1. How an answer is built (why the knowledge base matters)

```
question
  → scope guardrail        (off-topic / medical / unsafe → refuse, no model call)
  → hybrid retrieval       (Postgres full-text  ⊕  embedding cosine, fused via RRF)
  → live gym context       (branches, packages, facilities, FAQs, the member's own data)
  → model                  (strict system prompt: answer only from what you were given)
  → answer + citations
```

Everything the dashboard manages feeds step 2. If a member gets a vague answer,
the fix is almost always **a better document**, not a different prompt — and
`POST /sandy-ai/search` tells you exactly what Sandy was given.

### It ships already useful

`npm run sandy:seed` (idempotent) loads **27 curated documents** distilled from
public guidance by WHO, CDC, NHS, ACSM, NSCA, ACE, NASM, Harvard Health, Mayo
Clinic, Examine.com, Precision Nutrition, Stronger by Science and ExRx.net:
activity guidelines, progressive overload, rep ranges, RPE/RIR, programme
structure, warm-up, recovery, DOMS, protein, calories, macros, hydration,
creatine, supplements, sleep, spot-reduction, cardio, squat/deadlift/bench
technique, injury triage, women's strength training, training while fasting
(Ramadan), In-Body metrics, BMI, plateaus, gym etiquette.

---

## 2. Screens → endpoints

| Screen | Endpoint |
| --- | --- |
| Status card ("Sandy is online / not configured") | `GET /sandy-ai/health` |
| Stat cards | `GET /sandy-ai/stats` |
| Knowledge list + filters | `GET /sandy-ai/knowledge`, `GET /sandy-ai/categories` |
| "Add Document" (paste) | `POST /sandy-ai/knowledge` |
| "Import from URL" | `POST /sandy-ai/knowledge/ingest-url`, `GET /sandy-ai/sources` |
| Document detail / edit | `GET`/`PATCH`/`DELETE /sandy-ai/knowledge/:id` |
| "Test a question" panel | `POST /sandy-ai/search` |
| Member conversations | `GET /sandy-ai/conversations` |
| Transcript | `GET /sandy-ai/conversations/:id/messages` |
| Re-embed after config change | `POST /sandy-ai/knowledge/reindex` |

---

## 3. The status card — check this first

`GET /sandy-ai/health`:

```jsonc
{ "baseUrl": "https://api.openai.com/v1", "chatModel": "gpt-luna",
  "embeddingModel": "text-embedding-3-small",
  "chatConfigured": true, "embeddingConfigured": true,
  "retrievalMode": "hybrid" }
```

- `chatConfigured: false` → nobody can chat. Members still get `200` with
  `refusalReason: "PROVIDER_ERROR"`, so **the mobile app looks fine while Sandy
  is quietly dead** — this card is the only place staff will notice. Surface it
  prominently.
- `retrievalMode: "lexical-only"` → no embedding model configured. Retrieval
  still works (full-text) but won't match paraphrases. Worth a warning chip.
- The API key is never returned.

Sandy is provider-agnostic: it speaks the OpenAI-compatible format, so switching
providers is an `.env` change (`AI_BASE_URL`, `AI_API_KEY`, `AI_CHAT_MODEL`), not
a code change.

---

## 4. Adding knowledge

### Paste content — `POST /sandy-ai/knowledge`

```jsonc
{
  "title": "Glory Gym towel policy",
  "sourceName": "Glory Gym",
  "sourceUrl": "https://glorygym.com/policies",
  "category": "gym-info",
  "lang": "en",
  "searchAliases": "المنشفة، فوطة، سياسة المناشف",
  "content": "Members must bring their own towel..."
}
```

**`searchAliases` is the field to teach staff about.** It's indexed at *title
weight on every chunk* and is **never sent to the model**, so it adds no prompt
noise. Two uses:

- **Arabic terms for English content.** Most members type Arabic; most source
  material is English. Aliases are what let «كام جرام بروتين» retrieve an
  English protein document. Every curated doc ships with them.
- **Synonyms and slang** members actually use ("ديدليفت", "الرفعة الميتة").

Arabic text is normalised on both sides before indexing and searching —
diacritics and tatweel stripped, alef/ya/ta-marbuta forms unified, and the
definite article «ال» removed. So an alias written «التمرين» is found by a
member typing «تمرين» and vice versa; staff don't need to write both.

### Import from a URL — `POST /sandy-ai/knowledge/ingest-url`

Hosts are restricted to the registry from `GET /sandy-ai/sources` (17 sources,
tiered `guideline` / `professional` / `consumer`) — render it as the picker.
A host outside it returns `400` unless `allowAnyHost: true` is passed.

Re-ingesting an unchanged page returns `unchanged: true` and re-embeds nothing,
so a "refresh sources" button is cheap to offer.

The endpoint is SSRF-hardened: non-http(s) schemes and any host resolving to a
private/loopback/link-local address are rejected even with `allowAnyHost: true`
(re-checked across redirects), and the fetched body is size-capped. Re-ingest
only ever updates a previously *scraped* (WEB) doc — it never overwrites a
curated or manually-pasted document that happens to share the same URL.

> ⚠ Ingestion stores a **bounded excerpt with attribution and a link back**, not
> a full mirror of the article, and answers always cite the source. Worth
> keeping in mind before anyone asks for "import this whole site".

---

## 5. The retrieval preview — the debugging tool

`POST /sandy-ai/search` with `{ query, topK }` returns **exactly** the chunks
Sandy would be handed, with fused scores and which arm matched — and generates
nothing, so it costs no tokens:

```jsonc
[{ "chunkId": "...", "docId": "...", "title": "How much protein you actually need",
   "sourceName": "Examine.com", "content": "For people doing resistance training...",
   "score": 0.0328, "matchedBy": ["lexical", "vector"] }]
```

Build this into the admin UI as a "test a question" panel. When a member
complains about a bad answer, this shows whether the problem is *retrieval*
(wrong or no chunks came back → fix the document or its aliases) or
*generation* (right chunks, bad answer → a prompt/model issue).

An empty array means nothing cleared the relevance floor — Sandy would answer
from general knowledge and gym context only.

---

## 6. Oversight

`GET /sandy-ai/conversations` lists every member's threads (search matches
member name, member code, title). `GET /sandy-ai/conversations/:id/messages`
returns the transcript **oldest-first** — reading order, deliberately the
opposite of the member-side endpoint — including `model`, `promptTokens`,
`completionTokens` and `latencyMs` per assistant turn for cost/latency
monitoring.

`GET /sandy-ai/stats` drives the cards:

```jsonc
{ "knowledge": { "docs": 27, "activeDocs": 27, "chunks": 27,
                 "embeddedChunks": 27, "unembeddedChunks": 0 },
  "usage": { "conversations": 12, "messages": 84, "assistantMessages": 42, "refusals": 5 },
  "provider": { ... } }
```

A rising `refusals` count is worth showing — a spike usually means members are
asking for something Sandy has no documents for.

---

## 7. Gotchas

1. **`unembeddedChunks > 0`** means part of the knowledge base is invisible to
   semantic search. Happens after adding documents while the embedding model was
   unset, or after switching models. Fix: `POST /sandy-ai/knowledge/reindex`.
   It's the one Sandy endpoint that returns a hard `400` when no embedding model
   is configured.
2. **`PATCH` only re-chunks and re-embeds when `content` is included.** Editing
   just the title or category is cheap and leaves embeddings intact.
3. **Prefer `status: "DISABLED"` over delete.** A disabled document drops out of
   retrieval immediately but stays recoverable.
4. **Deleting a document deletes its chunks** (cascade) and its citations
   disappear from future answers — past messages keep their stored citations.
5. **The `search` filter on the knowledge list also matches chunk content**, not
   just the title, so results may look surprising until you open the document.
6. **`GET /sandy-ai/knowledge/:id` omits embedding vectors** by design — each
   chunk reports a boolean `embedded` instead. Vectors would be megabytes.
7. **Members and staff use different token families.** An employee token gets
   `401` on `/mobile/sandy/*` and vice versa.
