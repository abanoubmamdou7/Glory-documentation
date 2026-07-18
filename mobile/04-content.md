# 04 — Mobile Content ("معلومات عن جلوري جيم") + Feedback

> Audience: **mobile app**. The content reads are **public** (no token) —
> the terms/privacy pages are linked from the signup screen before login.
> Only feedback submission needs the member Bearer token.

Covers: the **معلومات عن جلوري جيم** menu (من نحن، تقيمات التطبيق، تواصل
معنا، قيمنا، رؤيتنا، أهدافنا، الشروط والأحكام، الأسئلة الشائعة، سياسة
الخصوصية) and **شكاوي و اقتراحات**.

| # | Method & path | Auth | Screen |
| --- | --- | --- | --- |
| 1 | `GET /mobile/content/pages` | Public | (prefetch all info pages) |
| 2 | `GET /mobile/content/pages/:key` | Public | Each info page |
| 3 | `GET /mobile/content/faqs` | Public | الأسئلة الشائعة |
| 4 | `GET /mobile/content/contact` | Public | منصات التواصل + تواصل معنا + تقيمات التطبيق |
| 5 | `POST /mobile/feedback` | **Member Bearer** | شكاوي و اقتراحات |

## 1–2. Info pages

`GET /mobile/content/pages` returns all six; `GET /mobile/content/pages/:key`
returns one (key is case-insensitive; unknown → `404`).

| `key` | Screen |
| --- | --- |
| `ABOUT` | من نحن (the "تعرف علي جلوري جيم" tab) |
| `VALUES` | قيمنا |
| `VISION` | رؤيتنا |
| `GOALS` | أهدافنا |
| `TERMS` | الشروط والأحكام |
| `PRIVACY` | سياسة الخصوصية |

```json
{ "success": true, "data": {
  "id": "...", "key": "TERMS",
  "titleEn": "Terms & Conditions", "titleAr": "الشروط والأحكام",
  "contentEn": "Subscription rules, ...",
  "contentAr": "قواعد الاشتراك، سياسة الإلغاء، قواعد استخدام الأجهزة، والمسؤولية القانونية.",
  "imageUrl": null, "updatedAt": "..." } }
```

Use `titleAr`/`contentAr` or the EN pair per app language; `imageUrl` is the
header photo (null until the gym uploads one — keep a local placeholder).
Content is plain text today (render line breaks).

## 3. `GET /mobile/content/faqs`

Active questions in display order:

```json
{ "success": true, "data": [ {
  "id": "...",
  "questionAr": "إزاي أقدر أشوف التمرينة اليومية بتاعتي؟",
  "answerAr": "من خلال علامة \"+\" في الصفحة الرئيسية",
  "questionEn": "How can I see my daily workout?",
  "answerEn": "Through the \"+\" sign on the home screen.",
  "sortOrder": 1 } ] }
```

Accordion UI; the "شكاوي و اقتراحات" button at the bottom opens the feedback
screen (endpoint 5).

## 4. `GET /mobile/content/contact`

Flat key→value map — drives **three** UI spots:

```json
{ "success": true, "data": {
  "social.whatsapp": "https://wa.me/2010...",
  "social.facebook": "https://facebook.com/glorygym",
  "social.instagram": "https://instagram.com/glorygym",
  "social.twitter": "https://x.com/glorygym",
  "contact.phone": "+2010...",
  "rating.appstore": "https://apps.apple.com/app/glory-gym",
  "rating.playstore": "https://play.google.com/store/apps/details?id=..."
} }
```

- **من نحن → منصات التواصل tab:** the `social.*` rows + `contact.phone`
  ("اتصل بنا" → `tel:` link).
- **تواصل معنا:** same data.
- **تقيمات التطبيق:** open `rating.appstore` / `rating.playstore` by platform.

Treat keys defensively (skip rows whose key you don't recognize — the gym may
add more).

## 5. `POST /mobile/feedback` — "شكاوي و اقتراحات"

Member Bearer token. Body: `{ "message": "..." }` (trimmed, 1–2000 chars —
disable إرسال while empty, like the design).

**201:** `{ "id": "...", "message": "...", "createdAt": "..." }` → show a
success state and clear the box. **400** blank/too long; **401** not logged in.

## Caching advice

Pages/FAQs/contact change rarely — cache them locally (e.g. refresh once per
app launch) and render instantly from cache. All three endpoints are cheap
and public, so no token juggling is needed for the pre-login terms/privacy
links.
