# 02 — Team Management (Admins / Staff / Instructors)

> Audience: **web dashboard frontend**. Covers the Team screens: the three
> list pages, the stats cards, CSV export, the 4-step **Add Employee wizard**,
> and the 4-tab employee profile.

The three roles share one identical API surface mounted three times:

```
{base} = /admins  |  /staff  |  /instructors
```

Everything documented once below works for all three. **Role scoping is
strict**: `GET /admins/:id` for a user who is actually STAFF → `404`.

### Permissions per endpoint

| Action | Key |
| --- | --- |
| All reads under `{base}` | `admins.read` / `staff.read` / `instructors.read` |
| All writes under `{base}` | `admins.manage` / `staff.manage` / `instructors.manage` |
| Catalogs (`/specialities`, `/permissions`, `/departments`, `/branches` reads) | any valid token |

ADMIN bypasses all keys.

### Endpoint index (15 per role + catalogs + upload)

| # | Method & path | Purpose |
| --- | --- | --- |
| 1 | `POST {base}` | Create employee — the whole wizard in one call |
| 2 | `GET {base}` | Paginated list (search/filter/sort) |
| 3 | `GET {base}/stats` | The 4 stat cards |
| 4 | `GET {base}/export` | CSV download |
| 5 | `GET {base}/next-employee-code` | Preview of the read-only Employee ID |
| 6 | `GET {base}/:id` | Fully-hydrated profile (all 4 tabs) |
| 7 | `PATCH {base}/:id` | Update profile/job info |
| 8 | `PATCH {base}/:id/password` | Set new password |
| 9 | `PATCH {base}/:id/status` | ACTIVE / INACTIVE / FREEZED |
| 10 | `PUT {base}/:id/documents` | Replace the document set |
| 11 | `DELETE {base}/:id/documents/:documentId` | Delete one document card |
| 12 | `PUT {base}/:id/shifts` | Replace the weekly schedule |
| 13 | `PATCH {base}/:id/shifts/:dayOfWeek` | Edit one weekday row |
| 14 | `PUT {base}/:id/permissions` | Set permissions |
| 15 | `DELETE {base}/:id` | Delete employee |
| — | `POST /uploads` | File upload (avatar / documents) |
| — | `GET /specialities` | Instructor speciality chips |
| — | `GET /permissions` | Permission catalog, grouped like the UI |

---

## File upload — `POST /uploads`

Multipart form, field name **`file`**. PNG / JPG / PDF, **max 5 MB**
(magic-byte checked server-side, so a renamed .txt still fails).

**201**

```json
{ "success": true, "data": {
  "url": "https://res.cloudinary.com/.../file.png",
  "publicId": "glory-gym/abc123", "format": "png", "mimeType": "image/png",
  "bytes": 327680, "fileSize": "320 KB", "originalName": "id-card.png"
} }
```

Use `data.url` for `avatarUrl` and `documents[].fileUrl`, and `data.fileSize`
for `documents[].fileSize`. Errors: `400` wrong type / too big / no file;
`503` Cloudinary env vars not configured on the server.

---

## The Add-Employee wizard → one `POST {base}`

The whole 4-step wizard submits as a **single call** on the final step. All
steps' data goes in one JSON body; the server creates user + branches +
specialities + documents + shifts + permissions **atomically** (all-or-nothing).

> ⚠ The Figma wizard has no password field, but the API **requires**
> `password` (min 8) — add a password input or generate one and show it.

### Step 1 — Profile Info

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `fullName` | string | **yes** | |
| `email` | string (email) | **yes** | Unique → `409` if taken |
| `password` | string | **yes** | min 8 |
| `username` | string | no | |
| `phoneCountryCode` | string | no | e.g. `"+966"` (the selector) |
| `phone` | string | no | |
| `gender` | `MALE` \| `FEMALE` | no | |
| `dateOfBirth` | date string | no | `"1995-05-20"` |
| `age` | int ≥ 0 | no | Or derive from DOB client-side |
| `nationality`, `nationalId`, `city`, `streetName` | string | no | |
| `avatarUrl` | string | no | From `POST /uploads` |

### Step 2 — Job Information

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `employeeCode` | string | no | **Omit it** — auto-generated (numeric, max+1, from 1000). Preview via `GET {base}/next-employee-code` → `{ "employeeCode": "1005" }` (non-reserving) |
| `departmentId` | id | no | From `GET /departments?limit=100` |
| `reportingToId` | user id | no | Compose the dropdown from the three role lists |
| `workLevel` | `JUNIOR`\|`MID`\|`SENIOR`\|`LEAD` | no | |
| `employmentType` | `FULL_TIME`\|`PART_TIME` | no | |
| `contractType` | `ANNUAL`\|`MONTHLY`\|`TEMPORARY` | no | |
| `contractStart`, `contractEnd` | date string | no | |
| `probationMonths` | int ≥ 0 | no | |
| `baseSalary` | **decimal string** | no | `"850.000"` — never a JS number |
| `primaryBranchId` | id | no | From `GET /branches` |
| `branchIds` | id[] | no | Branch links (unique ids) |

**Instructor-only extras:** `mainPayRateType`, `subPayRateType`, `serviceType`
(free strings), `specialityIds` (ids from `GET /specialities`).

### Step 3 — Documents

```json
"documents": [
  { "fileNameEn": "National ID Card", "fileNameAr": "البطاقة الشخصية",
    "fileUrl": "<from /uploads>", "fileSize": "320 KB" }
]
```

### Step 4 — Attendance (shifts) + Permissions

```json
"shifts": [
  { "dayOfWeek": "SUNDAY", "status": "WORKDAY", "startTime": "09:00 AM", "endTime": "05:00 PM" },
  { "dayOfWeek": "FRIDAY", "status": "DAY_OFF" }
],
"permissions": { "grantAll": true }
// or: "permissions": { "grantAll": false, "permissionKeys": ["staff.read", "dashboard.view"] }
```

Rules: max **one row per `dayOfWeek`** (`SUNDAY`…`SATURDAY`);
`WORKDAY` **requires** `startTime` + `endTime` (else `400`); `DAY_OFF` stores
null times even if sent. Times are plain strings — keep one format app-wide.

**201** → the fully-hydrated employee (same shape as `GET {base}/:id`).
Unknown permission key / bad branchId / bad specialityId → `400`.
Duplicate email → `409`.

---

## List page

### `GET {base}` — the table

Query: `page`, `limit`, `search`, `sortBy`, `sortOrder`, `status`
(`ACTIVE`|`INACTIVE`|`FREEZED`), `departmentId`.

- `search` matches: **id (exact)**, employeeCode (contains), name / phone /
  email (case-insensitive contains).
- `sortBy`: `id`, `employeeCode` (the ID Number column), `status`,
  `fullName`, `createdAt` (default).

Each item includes the scalars + `department`, `primaryBranch`, and
`_count.branches` — enough for every table column. No password, ever.

### `GET {base}/stats` — the 4 cards

```json
{ "success": true, "data": {
  "total":    { "count": 24, "changePct": 4.3 },
  "active":   { "count": 20, "changePct": 5.0 },
  "inactive": { "count": 3,  "changePct": 0 },
  "freezed":  { "count": 1,  "changePct": -50 }
} }
```

`changePct` = % change vs 7 days ago (approximated from `createdAt` — status
history isn't tracked).

### `GET {base}/export` — CSV button

Honors the same `search`/`status`/`departmentId` filters; **no pagination**.
Returns `text/csv` with `Content-Disposition: attachment` and a **UTF-8 BOM**
(Arabic-safe in Excel). This is a raw download — handle as a blob:

```ts
const res = await fetch(url, { headers: { Authorization: `Bearer ${t}` } });
const blob = await res.blob();
// createObjectURL + <a download> click
```

Columns: ID Number, Full Name, Username, Email, Phone (country code + phone
joined), Department, Primary Branch, Status, Joined At.

---

## Profile page (4 tabs)

### `GET {base}/:id`

One call feeds all tabs. Shape highlights:

```json
{
  "id": "...", "employeeCode": "1000", "fullName": "...", "email": "...",
  "role": "STAFF", "status": "ACTIVE", "avatarUrl": null,
  "department": { "id": "...", "name": "Sales" },
  "reportingTo": { "id": "...", "fullName": "..." },
  "primaryBranch": { "id": "...", "nameEn": "6th October" },
  "branches": [ { "id": "...", "nameEn": "..." } ],
  "specialities": [ { "id": "...", "name": "Personal Training" } ],
  "documents": [ { "id": "...", "fileNameEn": "...", "fileUrl": "...", "fileSize": "320 KB" } ],
  "shifts": [ { "dayOfWeek": "SUNDAY", "status": "WORKDAY", "startTime": "09:00 AM", "endTime": "05:00 PM" } ],
  "permissions": ["staff.read", "dashboard.view"],
  "baseSalary": "850.5"
}
```

> `baseSalary` normalizes trailing zeros (`"850.500"` → `"850.5"`). Exact
> value; format for display client-side.

### Editing

| UI action | Call |
| --- | --- |
| Edit profile/job fields | `PATCH {base}/:id` — send only changed fields. Include `branchIds`/`specialityIds` **only when changing them** (they **replace** the whole set) |
| Change password | `PATCH {base}/:id/password` `{ "password": "..." }` — revokes the user's sessions |
| Change status | `PATCH {base}/:id/status` `{ "status": "INACTIVE" }` — non-ACTIVE kills their access immediately |
| Documents tab — save all | `PUT {base}/:id/documents` `{ "documents": [...] }` (replaces the set) |
| Documents tab — delete one card | `DELETE {base}/:id/documents/:documentId` (`404` if it belongs to someone else) |
| Attendance — save week | `PUT {base}/:id/shifts` `{ "shifts": [...] }` (replaces; `{ "shifts": [] }` clears the week) |
| Attendance — edit one row | `PATCH {base}/:id/shifts/:dayOfWeek` (upserts that day; bad day name → `400`) |
| Permissions tab | `PUT {base}/:id/permissions` `{ "grantAll": ... , "permissionKeys": [...] }` |
| Delete employee | `DELETE {base}/:id` — **`409` if still referenced** elsewhere (show the message) |

---

## Catalogs for the wizard

### `GET /specialities`

```json
{ "success": true, "data": [ { "id": "...", "name": "EMS" }, { "id": "...", "name": "Personal Training" } ] }
```

### `GET /permissions` — grouped exactly like the design

```json
{ "success": true, "data": [
  { "group": "Settings", "total": 2,
    "permissions": [ { "key": "settings.view", "label": "View Settings" } ] },
  { "group": "Dashboard View and Setup", "total": 5, "permissions": [ ... ] }
] }
```

Render each group as a checkbox section with an "x / y Selected" badge from
`total`. Dropdown feeds: `GET /departments?limit=100` (paginated list of
`{ id, name }`) and `GET /branches` (paginated; use `nameEn`/`nameAr` + `id`).

---

## Gotchas checklist

1. `password` required on create but absent from the design (see above).
2. Never send `employeeCode` — preview it with `next-employee-code`; the real
   code is recomputed at create time (concurrency-safe).
3. `PATCH {base}/:id` with `branchIds`/`specialityIds` **replaces** those sets.
4. CSV export is not JSON — use blob handling; filename comes from
   `Content-Disposition`.
5. `stats.changePct` baseline is approximate (see above) — don't present it as
   exact history.
6. Employee delete can `409` — surface `message` ("still referenced").
7. Role scoping: an id from the Staff list will `404` on `/admins/:id`.
8. Decimal fields are strings; trailing zeros normalize in responses.
9. Times (`startTime`/`endTime`) are opaque strings — the API stores what you
   send. Standardize on one format (the design uses `"09:00 AM"`).
