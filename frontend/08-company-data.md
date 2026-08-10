# 08 — Company Data (Settings)

> Audience: **web dashboard frontend**. Covers the full **Settings →
> Company Data** sidebar — 9 tabs total. Reads need `settings.view`;
> writes need `settings.manage` (ADMIN bypasses both) **across all four
> resources on this page** (they all share the same two permission keys,
> even though they're four separate API resources under the hood).

The sidebar has **four different kinds** of tab, backed by four different
resources — don't assume they're all `company-pages`, and don't assume
"Contact Us" is a rich-text page like its neighbors:

| Sidebar tabs | Resource | Section below |
| --- | --- | --- |
| About Us, Our Vision, Our Mission, Our Goals, Terms & Conditions, Privacy Policy, Our Values | `/company-pages` (single page per tab, upsert) | §1 |
| Contact Us | `/contact-methods` (a **reorderable list** of contact buttons, full CRUD) | §2 |
| FAQ | `/faqs` (a list of Q&A cards, full CRUD) | §3 |
| Assessment Evaluation | `/assessment-questions` (a list of rating questions, full CRUD) | §4 |

All four are the **CMS backing the mobile app** — editing here is live,
with no cache/approval step. `/company-pages` feeds
`GET /mobile/content/pages`, `/contact-methods` feeds
`GET /mobile/content/contact`, `/faqs` feeds `GET /mobile/content/faqs`,
`/assessment-questions` feeds `GET /mobile/assessment/questions` (the
post-session "تقيم للحصه" rating flow — see
[../mobile/05-bookings-checkin.md](../mobile/05-bookings-checkin.md)).

---

## §1 — Content pages (`/company-pages`)

| # | Method & path | Screen |
| --- | --- | --- |
| 1 | `GET /company-pages/keys` | The sidebar tab list itself |
| 2 | `GET /company-pages` | (not directly one screen — see below) |
| 3 | `GET /company-pages/:key` | Clicking a sidebar tab — loads the edit form |
| 4 | `PATCH /company-pages/:key` | "Save Changes" |

### The sidebar

Don't hardcode the 7 tabs client-side — call `GET /company-pages/keys` and
render the sidebar from the response. It's a fixed catalog (order matches
the design), independent of which pages actually have saved content:

```json
[
  { "key": "ABOUT",   "labelEn": "About Us",             "labelAr": "من نحن" },
  { "key": "VISION",  "labelEn": "Our Vision",           "labelAr": "رؤيتنا" },
  { "key": "MISSION", "labelEn": "Our Mission",          "labelAr": "رسالتنا" },
  { "key": "GOALS",   "labelEn": "Our Goals",            "labelAr": "أهدافنا" },
  { "key": "TERMS",   "labelEn": "Terms & Conditions",   "labelAr": "الشروط والأحكام" },
  { "key": "PRIVACY", "labelEn": "Privacy Policy",       "labelAr": "سياسة الخصوصية" },
  { "key": "VALUES",  "labelEn": "Our Values",           "labelAr": "قيمنا" }
]
```

> `VALUES` ("قيمنا") isn't in this particular design's sidebar mockup — it
> predates this screen and is already live content the mobile app reads.
> Included here for completeness; drop it from your rendered sidebar if the
> real design genuinely excludes it, the API doesn't care either way.
>
> ⚠ **`CONTACT` is deliberately not a key here.** "Contact Us" first looked
> like it'd be another rich-text page, but the real design turned out to be
> a reorderable list of contact buttons — it lives at `/contact-methods`
> instead (§2). If you're wiring the "Contact Us" sidebar tab, point it at
> §2's endpoints, not `GET /company-pages/CONTACT` (that now 400s).

### Clicking a tab → `GET /company-pages/:key`

```json
{ "id": "...", "key": "VISION",
  "titleEn": "Our Vision", "titleAr": "رؤيتنا",
  "contentEn": "To be the leading fitness community in Jordan, inspiring life-changing transformations through elite coaching and unwavering discipline.",
  "contentAr": "أن نكون المجتمع الرياضي الرائد في الأردن، ...",
  "imageUrl": "https://res.cloudinary.com/.../vision.jpg",
  "updatedAt": "2026-08-09T07:56:14.832Z" }
```

- "Last updated: ..." on the page header = `updatedAt`, formatted client-side.
- **`404`** means this key has **never been saved** (e.g. `MISSION` on a
  fresh install) — this is not an error state. Render an **empty form**
  for that key (blank titles/content, no image) rather than showing an
  error toast; the first Save Changes on that form will create it.

### "Save Changes" → `PATCH /company-pages/:key`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `titleEn` | string | **yes** | |
| `titleAr` | string | **yes** | |
| `contentEn` | string | **yes** | Rich-text editor output (HTML/Markdown — stored as-is, format-agnostic) |
| `contentAr` | string | **yes** | Same |
| `imageUrl` | string \| `null` | no | From `POST /uploads` (PNG/JPG ≤5MB, matches the screen's own text); send `null` to remove the image |

**This is a full-page save, not a sparse patch** — the design's form always
shows both language editors together and the button submits the whole
visible form every time, so all four text fields are required on every
call even though the HTTP verb is `PATCH`. It's also an **upsert**: the
very first save for a key (e.g. `MISSION`) creates the row; every save
after that updates it — same endpoint, same request shape, no separate
"create" call needed.

**200:** the saved page (fresh `updatedAt`) → show the "Update Available —
[Page] information has been saved successfully" toast, then flip the
screen from edit mode back to the read-only **Edit** button state.

**Errors:**

| Code | Meaning | UI |
| --- | --- | --- |
| `400` | Unknown `:key`, or a required field missing/empty | Shouldn't happen from the fixed sidebar; show validation inline if it does |
| `403` | Missing `settings.manage` | Hide/disable the Save button for users without it |

### The Image box

Matches `POST /uploads` exactly (same 5MB / PNG+JPG rule the screen's
helper text states):

1. User drops/selects a file → `POST /uploads` (multipart, field `file`).
2. Response's `data.url` → hold it locally, then include as `imageUrl` in
   the next `PATCH /company-pages/:key` call.
3. The "eye" icon on the thumbnail just opens `imageUrl` (lightbox/new
   tab) — no API call.
4. The "✕" icon clears the local selection; on the next Save, send
   `imageUrl: null` to actually remove it server-side.

> ⚠ **Documented inference:** the design's dropzone says "Add **Another**
> Image" and a "1 images total" counter, which reads like multi-image
> support — but `AppInfoPage` stores a **single** `imageUrl` (schema
> field, not an array), and no screen shows more than one image attached
> at once. Treated as "add or replace the one image" — flag back if the
> real intent is a proper multi-image gallery, since that would need a
> schema change (a new related table) beyond this task's scope.

### "Not saved yet, click to save" banner / Cancel button

Both are **purely client-side** — no API involvement:
- The banner appears whenever local form state differs from the last
  fetched `GET` response (a dirty-check).
- **Cancel** just discards local edits and re-renders from the last
  successful `GET /company-pages/:key` (no need to re-fetch unless you
  want to be extra safe against concurrent edits).

---

## §2 — Contact Us (`/contact-methods`)

**Not** a single EN/AR+image page — the "Contact Us" screen manages a
**reorderable list of contact buttons** (WhatsApp, phone, email, social
links, store links), each with an icon picked from a fixed set. Matches
the "Edit Contact Method" modal and the draggable card list exactly.

| # | Method & path | Screen |
| --- | --- | --- |
| 1 | `POST /contact-methods` | "Add Another Contact Method" |
| 2 | `GET /contact-methods` | The draggable list |
| 3 | `GET /contact-methods/:id` | Prefills "Edit Contact Method" |
| 4 | `PATCH /contact-methods/:id` | "Edit Method" save |
| 5 | `PUT /contact-methods/reorder` | After a drag-and-drop |
| 6 | `DELETE /contact-methods/:id` | The trash icon on a card |

### `POST /contact-methods` / `PATCH /contact-methods/:id`

| Field | Type | Required on create | Screen label |
| --- | --- | --- | --- |
| `icon` | enum, see below | **yes** | "Set Icons" grid |
| `labelEn` | string | **yes** | "Platform Name (EN)" |
| `labelAr` | string | **yes** | "Platform Name (Ar)" |
| `value` | string | **yes** | "Value / Link" (URL, phone number, or email — whatever fits the icon) |
| `sortOrder` | int | no | Omit on create to append to the end of the list |

`icon` — exactly the 10 boxes in the "Set Icons" picker, no free typing:
`WHATSAPP`, `INSTAGRAM`, `FACEBOOK`, `X`, `EMAIL`, `PHONE`, `WEBSITE`,
`GOOGLE_PLAY`, `APP_STORE`, `OTHER`. Map each to its icon asset client-side.

`PATCH` is a **true partial update** — send only what changed (e.g. just
`{ "value": "..." }` to fix a broken link without touching the label/icon).

### `GET /contact-methods` — the list

Returns every row already sorted by `sortOrder` ascending — render top to
bottom as-is, no client-side sorting needed:

```json
[
  { "id": "...", "icon": "GOOGLE_PLAY", "labelEn": "Google Play", "labelAr": "جوجل بلاي",
    "value": "https://play.google.com/store/apps/details?id=...", "sortOrder": 0, ... },
  { "id": "...", "icon": "EMAIL", "labelEn": "Email Support", "labelAr": "الدعم عبر البريد",
    "value": "support@glorygym.com", "sortOrder": 1, ... }
]
```

### `PUT /contact-methods/reorder` — after a drag

Send the drag handle's final order as a **flat array of every row's id**
(the whole list, not just the moved item):

```json
{ "orderedIds": ["id-of-row-now-first", "id-of-row-now-second", "..."] }
```

The array must contain **exactly** the ids currently in the list — no
more, no fewer, no unknowns — just permuted. The server sets each row's
`sortOrder` to its index in the array and returns the freshly-sorted list.
**`400`** if the id set doesn't match exactly (e.g. you fetched the list,
someone else deleted a row, then you tried to reorder using the stale
set) — on that error, just re-fetch `GET /contact-methods` and retry.

### Delete

`DELETE /contact-methods/:id` — plain removal, no confirmation dance
needed server-side (nothing else references a contact method).

---

## §3 — FAQ (`/faqs`)

A flat list of Q&A cards — full CRUD, same list-CRUD shape as Contact
Methods, minus the reorder endpoint (no drag handles shown on this screen).

| # | Method & path | Notes |
| --- | --- | --- |
| 1 | `POST /faqs` | Add a card |
| 2 | `GET /faqs` | The list |
| 3 | `GET /faqs/:id` | One card |
| 4 | `PATCH /faqs/:id` | Edit / reorder / toggle active |
| 5 | `DELETE /faqs/:id` | Remove a card |

### `POST /faqs` / `PATCH /faqs/:id`

| Field | Type | Required on create | Notes |
| --- | --- | --- | --- |
| `questionEn` | string | ✅ | |
| `questionAr` | string | ✅ | |
| `answerEn` | string | ✅ | |
| `answerAr` | string | ✅ | |
| `sortOrder` | int | no | Lower shows first in the mobile FAQ list |
| `isActive` | boolean | no (defaults `true`) | `false` hides it from the mobile app immediately |

`PATCH` is a **true partial update** here (unlike `/company-pages/:key`) —
send only the fields you're changing, e.g. `{ "isActive": false }` to
retire a FAQ without touching its text. (There's no dedicated reorder
endpoint here — set `sortOrder` directly per row via `PATCH`.)

### `GET /faqs`

Standard pagination + `search` (matches `questionEn`/`questionAr`) +
`isActive` filter. Default sort is `sortOrder` ascending (matches the
mobile display order) unless you pass `sortBy=questionEn`.

---

## §4 — Assessment Evaluation (`/assessment-questions`)

The rating questions shown on the mobile app's post-session "تقيم للحصه"
screen (one 1-5 star row per active question). Same list-CRUD shape as §3.

| # | Method & path | Notes |
| --- | --- | --- |
| 1 | `POST /assessment-questions` | Add a question |
| 2 | `GET /assessment-questions` | The list |
| 3 | `GET /assessment-questions/:id` | One question |
| 4 | `PATCH /assessment-questions/:id` | Edit / reorder / toggle active |
| 5 | `DELETE /assessment-questions/:id` | Remove — **see the 409 case below** |

### `POST /assessment-questions` / `PATCH /assessment-questions/:id`

| Field | Type | Required on create | Notes |
| --- | --- | --- | --- |
| `questionEn` | string | ✅ | |
| `questionAr` | string | ✅ | |
| `sortOrder` | int | no | Lower shows first in the rating form |
| `isActive` | boolean | no (defaults `true`) | `false` excludes it from **new** ratings |

`PATCH` is a true partial update, same as FAQ.

### ⚠ Deleting a question that's already been used

If a member has already rated a session using this question,
`DELETE /assessment-questions/:id` returns **`409`**:

```json
{ "success": false, "message": "This question has already been answered in past ratings and cannot be deleted — set isActive to false to retire it instead", "statusCode": 409 }
```

This is by design — deleting it would corrupt historical rating data.
**Handle this in the UI**: if delete fails with 409, offer "retire it
instead" which just calls `PATCH .../:id { "isActive": false }`. A
retired question stays visible in past ratings' detail views but won't
appear in new "تقيم للحصه" forms.

### `GET /assessment-questions`

Same shape as FAQ's list: pagination + `search` + `isActive` filter,
default sort `sortOrder` ascending.

---

## Gotchas checklist

1. `GET /company-pages/:key` returning `404` is the **normal** state for
   `MISSION` before anyone has saved it — not a bug.
2. `CONTACT` is **not** a `/company-pages` key — "Contact Us" is its own
   resource, `/contact-methods` (§2).
3. `/company-pages`' `PATCH` requires all four text fields every call
   (whole-page save); `/contact-methods`, `/faqs`, and
   `/assessment-questions` are **true partial** PATCHes — send only what
   changed. Don't mix up the two patterns.
4. `imageUrl: null` (explicit null) removes a company-page's image;
   omitting the field entirely also works but is less explicit — prefer
   sending `null`.
5. Reads and writes use **different** permission keys (`settings.view` vs
   `settings.manage`) across **all four** resources on this page — a user
   can be read-only here.
6. Editing anywhere on this page is **live** — the mobile app has no
   cache/approval step; every save is immediately visible to members.
7. `?isActive=false` on the FAQ/Assessment list endpoints really does
   filter to inactive rows — no special-casing needed on your end.
   (Flagging because a server-side bug briefly made this filter ignore
   `false` entirely; already fixed and verified before this doc shipped.)
8. `PUT /contact-methods/reorder` needs the **complete** current id set,
   not just the ids that moved — re-fetch the list first if you're not
   sure it's fresh.
