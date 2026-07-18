# 03 — Mobile Family ("أفراد العائلة")

> Audience: **mobile app**. Member Bearer token required on every call.
> Covers the family list screen and the **اضافة فراد للعائلة** form.

Base path: `/mobile/family`. Every row is **scoped to the logged-in member**:
another member's row (or an unknown id) behaves as if it doesn't exist → `404`.

| # | Method & path | Screen element |
| --- | --- | --- |
| 1 | `GET /mobile/family` | The cards list |
| 2 | `POST /mobile/family` | "أضافة عضو جديد" |
| 3 | `PATCH /mobile/family/:id` | Card "..." menu → edit |
| 4 | `DELETE /mobile/family/:id` | Card "..." menu → delete |

## 1. `GET /mobile/family`

Paginated (`page`/`limit`), newest first.

```json
{ "success": true,
  "data": [ {
    "id": "cm...",
    "fullName": "احمد حسام",                        // card title
    "relation": "SON",                              // العلاقة (ابن)
    "gender": "MALE",
    "dateOfBirth": "2015-06-10T00:00:00.000Z",      // تاريخ الميلاد
    "createdAt": "2026-06-10T09:00:00.000Z",        // تاريخ الاضافة
    "email": null, "phone": null, "memberId": "cm..."
  } ],
  "meta": { "page": 1, "limit": 10, "total": 1, "totalPages": 1 } }
```

Card date formatting (e.g. "١٠ يونيو ٢٠٢٦", "يونيو") is client-side.

## 2. `POST /mobile/family` — the add form

| Field | Type | Required | Screen label |
| --- | --- | --- | --- |
| `fullName` | string ≤120 | **yes** | الاسم الكامل |
| `dateOfBirth` | date string `"2015-06-10"` | no | تاريخ الميلاد |
| `gender` | `MALE` \| `FEMALE` | no | النوع |
| `relation` | `SON` \| `DAUGHTER` \| `SPOUSE` \| `OTHER` | **yes** | العلاقة |

**201:** the created row (same shape as the list item).

**400 examples:** missing name; `relation must be one of the following
values: SON, DAUGHTER, SPOUSE, OTHER`; bad date. Map the relation enum to the
dropdown: ابن=SON، ابنة=DAUGHTER، زوج/زوجة=SPOUSE، أخرى=OTHER.

## 3. `PATCH /mobile/family/:id`

Same fields as create, all optional — send only the changes.
**200:** the updated row. **404:** not found / not yours.

## 4. `DELETE /mobile/family/:id`

**200:** `{ "message": "Family member removed" }`. **404:** already deleted /
not yours — safe to treat as success + refresh the list.

## Notes

- Family members here are **profile records only** (no accounts/logins).
  Package family-sharing rules come later with the sales phases.
- They are deleted automatically if the member deletes their account.
