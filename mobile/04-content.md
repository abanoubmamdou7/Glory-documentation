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
| `MISSION` | رسالتنا |
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

An **ordered array** (not a key-value map) — the gym manages this list from
the dashboard's Company Data → "Contact Us" screen, including drag-to-reorder,
so the set of rows and their order can change at any time:

```json
{ "success": true, "data": [
  { "id": "...", "icon": "WHATSAPP", "labelEn": "Whatsapp", "labelAr": "واتساب", "value": "https://wa.me/2010..." },
  { "id": "...", "icon": "INSTAGRAM", "labelEn": "Instagram", "labelAr": "انستجرام", "value": "https://instagram.com/glorygym" },
  { "id": "...", "icon": "FACEBOOK", "labelEn": "Facebook", "labelAr": "فيسبوك", "value": "https://facebook.com/glorygym" },
  { "id": "...", "icon": "X", "labelEn": "X", "labelAr": "إكس", "value": "https://x.com/glorygym" },
  { "id": "...", "icon": "PHONE", "labelEn": "Phone Number", "labelAr": "رقم الهاتف", "value": "+2010..." },
  { "id": "...", "icon": "EMAIL", "labelEn": "Email Support", "labelAr": "الدعم عبر البريد", "value": "support@glorygym.com" },
  { "id": "...", "icon": "APP_STORE", "labelEn": "App Store", "labelAr": "آب ستور", "value": "https://apps.apple.com/app/glory-gym" },
  { "id": "...", "icon": "GOOGLE_PLAY", "labelEn": "Google Play", "labelAr": "جوجل بلاي", "value": "https://play.google.com/store/apps/details?id=..." }
] }
```

Renders the same array across **all three** UI spots:
- **من نحن → منصات التواصل tab** and **تواصل معنا:** iterate the array,
  pick the icon asset matching `icon`, use `labelAr`/`labelEn` as the row
  title and `value` as the tappable link — `PHONE`/`EMAIL` rows open
  `tel:`/`mailto:`, everything else opens as a URL.
- **تقيمات التطبيق:** filter for `icon === "APP_STORE"` / `"GOOGLE_PLAY"`
  and open the matching `value`.

`icon` is one of `WHATSAPP`|`INSTAGRAM`|`FACEBOOK`|`X`|`EMAIL`|`PHONE`|
`WEBSITE`|`GOOGLE_PLAY`|`APP_STORE`|`OTHER` — map each to a local asset;
fall back to a generic icon for `OTHER`/anything unrecognized (the gym may
add more rows with icons your build doesn't know about yet).

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
