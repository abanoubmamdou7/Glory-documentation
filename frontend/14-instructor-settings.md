# 14 — Settings: Profile Management (Instructor Dashboard, isolated)

> Audience: **the Instructor Dashboard frontend specifically** — a
> self-contained reference for the "Settings > Profile Management" screen,
> so this team doesn't need to also read the general dashboard's
> [01-auth.md](01-auth.md). Every endpoint below is identical to that
> doc's §5–7 — this is the same API, just framed for one caller: **you are
> always managing your own account**. Bearer token required on every
> endpoint. **No `*.read`/`*.manage` permission is needed for any of
> these** — unlike Workouts/Members/Follow-up Programs, there's no
> "Getting access" step here. You can always see and edit yourself, full
> stop, regardless of what's been granted to your account.

---

## The screens, top to bottom

| # | Screen | Call |
| --- | --- | --- |
| 1 | Profile Info / Job Information / Documents / Attendance tabs | `GET /auth/me/profile` |
| 2 | "Edit Profile" | `PATCH /auth/me/profile` |
| 3 | "Update Password" | `PATCH /auth/me/password` |

---

## 1. `GET /auth/me/profile`

One call loads **all four tabs** — pick which fields to render per tab
client-side, no separate request per tab:

```json
{ "id": "...", "role": "INSTRUCTOR", "status": "ACTIVE",
  "username": "Saudi", "fullName": "Ahmed Hossam",
  "email": "Ahmed.Hossam@glorygym.com",
  "phoneCountryCode": "+966", "phone": "01023359621",
  "gender": "MALE", "dateOfBirth": "2003-09-19T...", "age": 23,
  "nationality": "Egyptian", "nationalId": "3030919203333",
  "city": "6th October", "streetName": "Street Name", "avatarUrl": null,

  "employeeCode": "3522", "departmentId": "...", "department": { "id": "...", "name": "General Manager" },
  "reportingToId": "...", "reportingTo": { "id": "...", "fullName": "Bardees Banat" },
  "workLevel": "SENIOR", "employmentType": "PART_TIME",
  "contractType": "ANNUAL", "contractStart": "2024-01-15T...", "contractEnd": "2025-01-14T...",
  "probationMonths": 3, "baseSalary": "12000",
  "primaryBranchId": "...", "branches": [{ "id": "...", "nameEn": "6th October" }],

  "documents": [
    { "id": "...", "fileNameEn": "National ID Card", "fileUrl": "...", "fileSize": "320 KB", "createdAt": "2024-01-15T..." }
  ],

  "shifts": [
    { "dayOfWeek": "SUNDAY", "status": "WORKDAY", "startTime": "09:00 AM", "endTime": "05:00 PM" },
    { "dayOfWeek": "FRIDAY", "status": "DAY_OFF", "startTime": null, "endTime": null }
  ],

  "permissions": ["followup.manage", "workouts.manage"] }
```

Mapping: **Profile Info tab** = the top block (`username` through
`avatarUrl`). **Job Information tab** = `employeeCode` ("Employee ID"),
`department`/`reportingTo` (already resolved to name, not just an id),
`workLevel`, `employmentType`, `contractType`/`contractStart`/
`contractEnd`, `probationMonths`, `baseSalary`, `branches` ("Branch
Location" — join the names). The **"Staff Permissions"** block on this
same tab is **display-only** here — render `permissions` as the checked
items in the (read-only) group list from `GET /permissions` (see
[02-team-management.md](02-team-management.md)); there is no self-service
way to change your own permissions, and there shouldn't be. **Documents
tab** = `documents[]` — "View" opens `fileUrl` directly, "Download" is the
same URL with a download attribute/header on your end. **Attendance tab**
= `shifts[]`, one row per `DayOfWeek`; `DAY_OFF` rows have `startTime`/
`endTime: null` → render "No Shift Scheduled".

`password` is never included in the response, obviously.

## 2. `PATCH /auth/me/profile` — "Edit Profile"

Send only the fields the user actually changed (all optional):

```json
{
  "fullName": "Ahmed Hossam",
  "phoneCountryCode": "+966",
  "phone": "01023359621",
  "gender": "MALE",
  "dateOfBirth": "2003-09-19",
  "age": 23,
  "nationality": "Egyptian",
  "nationalId": "3030919203333",
  "city": "6th October",
  "streetName": "Street Name",
  "avatarUrl": "https://..."
}
```

`avatarUrl` comes from `POST /uploads` (upload the image first, then send
the returned URL here — same pattern as everywhere else in this API).

**This endpoint only accepts Profile Info fields.** Job Information
(department, reporting-to, work level, contract, salary, branches),
Documents, Shifts/Attendance, and Permissions are **not editable through
this screen** — those stay controlled by an ADMIN/STAFF with
`instructors.manage` via the separate Team Management API. `email` isn't
editable here either — there's currently no self-service email-change
flow for employees.

**Success `200`:** the updated profile, same shape as §1.

**Errors:** `400` if you send a field this DTO doesn't accept (job
info, `email`, `permissions`, etc.) — the request is rejected outright,
not silently partially applied. Don't build an "Edit Profile" form that
also tries to submit Job Information fields through this call.

## 3. `PATCH /auth/me/password` — "Update Password"

```json
{
  "currentPassword": "OldPass1122#",
  "password": "NewPass1122#",
  "passwordConfirm": "NewPass1122#"
}
```

All three required. New password: min 8 characters, at least one digit,
at least one letter.

**Success `200`:** `{ "id": "..." }`. This **logs you out everywhere** —
every refresh token you hold is revoked, so the very next token refresh
(yours included, on whatever device made this call) will fail. Route
straight to the login screen after a successful response, don't try to
keep the current session alive.

**Errors:** `400` — either `currentPassword` doesn't match your actual
current password ("Current password is incorrect"), or `password` !=
`passwordConfirm` ("Passwords do not match"). Show these inline on the
respective fields.

---

## Gotchas checklist

1. `GET /auth/me/profile` needs **no permission** — don't gate the
   Settings nav item behind `instructors.read` or similar; every logged-in
   employee can always see their own page.
2. `PATCH /auth/me/profile` is Profile-Info-only. Don't wire the Job
   Information tab's fields into this call — there's no edit action for
   that tab in this API.
3. "Staff Permissions" on the Job Information tab is **read-only** here —
   don't build a checkbox UI that tries to PATCH it; it doesn't exist on
   this endpoint by design.
4. `PATCH /auth/me/password` requires the **current** password — this is
   different from how an admin resets someone else's password in Team
   Management (no current-password check there). Don't reuse that other
   request shape.
5. A successful password change ends the current session — always
   redirect to login afterward, don't assume the existing access token
   keeps working past its own expiry.
6. The mockup's profile-card header text ("Kristina Gislason") doesn't
   match the actual data anywhere else on the screen — it's a stale
   reused component; always render the header from the `GET`
   response, never the mockup's literal name.
