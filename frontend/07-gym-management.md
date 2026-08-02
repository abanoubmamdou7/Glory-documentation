# 07 — Gym Management (Products & Push Notifications)

> Audience: **web dashboard frontend**. Covers the **Gym Management** nav
> section's screens: the **Products** list + **Add New Product** form, and
> **Push Notifications Management** + **Push New Notification**. (The third
> item, **Branchs**, is documented in the archived organization guide — its
> API is unchanged.)
>
> Reads need only a valid employee token; writes need `products.manage` /
> `notifications.manage` respectively (ADMIN bypasses).

| # | Method & path | Screen |
| --- | --- | --- |
| 1 | `GET /products` | Products list + search/filters |
| 2 | `GET /products/categories` | Category filter dropdown |
| 3 | `GET /products/export` | Export button (CSV) |
| 4 | `POST /products` | Add New Product — "Add Product" |
| 5 | `GET /products/:id` | Prefill edit / eye action |
| 6 | `PATCH /products/:id` | Edit product |
| 7 | `DELETE /products/:id` | Trash action |
| 8 | `POST /push-notifications` | Push New Notification — "Send Now" |
| 9 | `GET /push-notifications` | Notifications table |
| 10 | `GET /push-notifications/export` | Export button (CSV) |
| 11 | `GET /push-notifications/:id` | Row detail |
| — | `POST /uploads` | Product image + notification image |

---

## Products

### 1. `GET /products` — the list

Query: `page`, `limit`, `search` (matches **id exact** or **name**
contains), `productType`, `category`, `branchId`, `sortBy`
(`name`|`retailPrice`|`createdAt`), `sortOrder`.

Column mapping: "ID Number" = `id`; "Product Details" = `imageUrl` thumbnail +
`name`; "Product Type" = `productType`; "Category" = `category`;
"Retail Price" = `retailPrice` + " JOD" (decimal **string** — trailing zeros
normalize, e.g. `"29.990"` → `"29.99"`; format client-side). The header
count "( 123 )" = `meta.total`.

### 2. `GET /products/categories`

`{ "data": ["Nutrition", "Store", …] }` — distinct values in use, sorted.
Feeds the Category filter dropdown (and the form's Category field — free
text, so allow typing new ones too, same pattern as the Video Library).

### 3. `GET /products/export` — Export button

Returns raw `text/csv` (UTF-8 BOM, Arabic-safe in Excel) honoring the same
`search`/`productType`/`category` filters currently applied to the list —
**not JSON-enveloped**; handle as a blob download. Columns: ID, Name,
Product Type, Category, Retail Price, Tax, Weight, Weight Unit, Number of
Days, Pay Type, Branch, Created At.

### 4. `POST /products` — "Add New Product"

| Field | Type | Required | Screen label |
| --- | --- | --- | --- |
| `name` | string ≤150 | **yes** | Product Name |
| `imageUrl` | URL | no | The image circle — from `POST /uploads` |
| `productType` | string | no | Product Type |
| `retailPrice` | decimal string `"29.990"` | **yes** | Retail Price |
| `tax` | decimal string `"14.00"` | no | Tax |
| `category` | string | no | Category |
| `weight` | decimal string `"2.00"` | no | Weight |
| `weightUnit` | string | no | Weigh Unit |
| `numberOfDays` | int ≥ 0 | no | Number of days |
| `payType` | `ONE_TIME` \| `RECURRING` | no | Product Type pay |
| `branchId` | id | no | (validated to exist → `400`) |

Upload the image first via `POST /uploads` (multipart field `file`, PNG/JPG
≤5MB) → pass `data.url` as `imageUrl`. **201:** the created product.

> Design note: the mock's Weight field shows a "Number of days" placeholder —
> that's a Figma copy-paste slip; `weight` and `numberOfDays` are separate
> fields in the API.

### 5–7. Get / edit / delete

Standard: `GET /products/:id` prefills the edit form; `PATCH` takes any
create field (send only changes); `DELETE` returns `{ id }` (`404` if gone).

---

## Push Notifications Management

### 8. `POST /push-notifications` — "Send Now"

One call creates the notification **and broadcasts it**: a
`MemberNotification` inbox row is created for every targeted member,
atomically — this is exactly what the mobile app's "الاشعارات" inbox reads,
so the message appears in members' apps immediately.

| Field | Type | Required | Screen label |
| --- | --- | --- | --- |
| `titleEn` | string ≤150 | **yes** | Notification Title (EN) |
| `titleAr` | string ≤150 | no | Notification Title (AR) |
| `bodyEn` | string | no | Message Body (EN) |
| `bodyAr` | string | no | Message Body (AR) |
| `type` | `PUSH` \| `REMINDER` \| `AUTO` | **yes** | Notification Types |
| `targetAudience` | `ALL_MEMBERS` \| `ACTIVE_MEMBERS` \| `MALE_MEMBERS` \| `FEMALE_MEMBERS` | **yes** | Target Audience |
| `imageUrl` | URL | no | Image drop-zone — from `POST /uploads` (PNG ≤5MB) |

**201:** the notification with `dateSent`, `createdBy` (Send By), and the
four counters.

> ⚠ **Two documented inferences to verify against final designs:**
> 1. The Target Audience dropdown options aren't specified in the mock — the
>    four values above are the implemented set; more (e.g. per-branch) can be
>    added later.
> 2. The message-body toolbar suggests rich text, but the API stores plain
>    text (`bodyEn`/`bodyAr`) — the mobile inbox renders plain text too. If
>    rich text is really needed, that's a schema-level change to request.

> **Delivery counters:** there's no FCM/device-token infra yet (Phase 12), so
> `totalCount` = targeted members, `successCount` = those with push enabled,
> `noTokenCount` = those with it disabled, `failedCount` = 0 — the "Total /
> Success / Failed / No Token" cells map straight onto these. The inbox
> delivery itself is real.

### 9. `GET /push-notifications` — the table

Query: `page`, `limit` (the "Row Per Page" dropdown), `search` (matches
title/body EN+AR **and** the sender's name), `type`, `sortBy`
(`dateSent` default desc | `sendBy`), `sortOrder`.

Column mapping: "Notifications Details" = `imageUrl` + `titleEn`/`titleAr` +
body preview; "Date Sent" = `dateSent`; "Notification Types" = `type`
(Push/Reminder/Auto); "Send By" = `createdBy.fullName`; "Target Audience"
cell = the four counters (`totalCount`/`successCount`/`failedCount`/
`noTokenCount`).

### 10. `GET /push-notifications/export`

Raw CSV (same blob handling as products), honoring `search`/`type`. Columns
include the four counters.

### 11. `GET /push-notifications/:id`

One row with counters + sender — for a detail view. `404` if not found.

---

## Gotchas checklist

1. Both image fields (product + notification) come from the existing
   `POST /uploads` (Cloudinary) — upload first, then pass the URL. No new
   upload endpoint.
2. Exports are raw CSV, not the JSON envelope — blob-download them.
3. There is **no edit/delete for sent notifications** — a broadcast that
   already reached member inboxes can't be unsent; the design shows no such
   action either.
4. Decimal fields are strings in and out (trailing zeros normalize).
5. To demo Send Now end-to-end, run `npm run seed:test` first so members
   exist, then log into the mobile side as `member1@glorytest.local` /
   `Test1234` and watch the inbox update.
