# Audit log (dashboard)

Everything an employee does on the dashboard is recorded automatically: who,
what, which record, when, from where, and **which fields changed, from what to
what**. The frontend only **reads** the log. Nothing is sent to "write" an
entry, and there is no endpoint to edit or delete one.

Base URL: same API. Auth: the employee bearer token.

| Screen | Endpoint | Permission |
| --- | --- | --- |
| Audit log table | `GET /audit-logs` | `audit.view` ("View Audit Log") |
| Entry details (drawer / page) | `GET /audit-logs/:id` | `audit.view` |
| Cards + charts at the top | `GET /audit-logs/stats` | `audit.view` |
| Filter dropdowns | `GET /audit-logs/filters` | `audit.view` |
| "History" tab on any record | `GET /audit-logs/entity/:entityType/:entityId` | `audit.view` |
| Export button | `GET /audit-logs/export?format=csv\|xlsx` | `audit.export` ("Export Audit Log") |
| "My activity" (every employee) | `GET /audit-logs/me` | none |

ADMIN sees everything. Both permissions are in `GET /permissions` under the
group **Audit Log**; grant them from the employee's permissions screen.

## 1. What gets recorded

| Recorded | Not recorded |
| --- | --- |
| Every create, edit, delete and action (freeze, void, approve, activate…) on dashboard routes | Ordinary page loads (`GET`), unless the server sets `AUDIT_LOG_READS=true` |
| Sign-in, failed sign-in (listed under the account that was tried), sign-out, password changes | Token refreshes, `GET /auth/me` |
| Every export / download (`…/export`, PDFs, templates) | The member mobile app (`/mobile/*`) |
| Requests the employee was **refused** (403) and requests that **failed** (400/409/500…) | Requests with an expired or missing token (not an action anyone took) |
| | Webhooks, health checks |

Passwords, tokens, OTP codes and secrets are **never stored**: they appear as
`"[REDACTED]"`. A password change shows that the password changed, never the
value.

## 2. The entry object

`GET /audit-logs` returns the light version. `GET /audit-logs/:id` adds
`changes`, `requestBody`, `query`, `before`, `route` and `userAgent`.

```json
{
  "id": "cmwt1e3ia000c7d7mvqk5cqpe",
  "createdAt": "2026-11-23T09:01:41.315Z",
  "requestId": "266e370d-949a-4f8c-b828-cf19a2cf25e6",
  "actor": { "id": "cm…", "name": "Test Admin", "email": "admin1@…", "role": "ADMIN" },
  "actorId": "cm…", "actorName": "Test Admin", "actorEmail": "admin1@…", "actorRole": "ADMIN",
  "branchId": "cm…",
  "action": "UPDATE",
  "operation": "PackagesController.update",
  "module": "packages",
  "moduleLabelEn": "Package",
  "moduleLabelAr": "باقة",
  "entityType": "Package",
  "entityId": "cmwt1e3h400087d7mcurl6nil",
  "entityLabel": "Gold 3 Months Promo",
  "summaryEn": "Test Admin updated package “Gold 3 Months Promo”",
  "summaryAr": "Test Admin عدّل باقة “Gold 3 Months Promo”",
  "method": "PATCH",
  "path": "/packages/cmwt1e3h400087d7mcurl6nil",
  "statusCode": 200,
  "outcome": "SUCCESS",
  "severity": "INFO",
  "errorMessage": null,
  "durationMs": 26,
  "ip": "203.0.113.24",
  "client": { "browser": "Chrome 131", "os": "macOS", "device": "desktop" },
  "changedFields": ["nameEn", "price"]
}
```

Detail only:

```json
{
  "route": "/packages/:id",
  "userAgent": "Mozilla/5.0 (Macintosh; …) Chrome/131.0.0.0 Safari/537.36",
  "query": null,
  "requestBody": { "price": "85.000", "nameEn": "Gold 3 Months Promo" },
  "changes": [
    { "field": "nameEn", "from": "Gold 3 Months", "to": "Gold 3 Months Promo" },
    { "field": "price", "from": "90", "to": "85" }
  ],
  "before": null
}
```

| Field | Meaning |
| --- | --- |
| `summaryEn` / `summaryAr` | Ready-to-show sentence. Show it as the row's main text. |
| `actor` | Who did it. `null` only for a failed sign-in with an email that matches no account (then `actorEmail` holds what was typed). Names are a **snapshot**: they stay right even if the employee is renamed or removed. |
| `action` | See §5. |
| `module` + `moduleLabelEn/Ar` | Dashboard area (first part of the URL). |
| `entityType` / `entityId` / `entityLabel` | The record acted on, e.g. `Member` / id / "Sara Ali". Can be `null` for actions that aren't about one record (a report export, a checkout). |
| `changes` | Field-by-field difference, before → after. On a create, `from` is `null`. Empty `[]` when nothing changed or the area has no record snapshot. |
| `changedFields` | Just the field names (list rows use this). |
| `before` | The full record as it was, **only on a delete**. |
| `requestBody` | What the dashboard sent, secrets hidden. File uploads show `{ file: { name, type, sizeBytes } }`. |
| `outcome` | `SUCCESS` (status < 400) or `FAILURE`. |
| `severity` | `INFO`, `WARNING` (deletes, cancellations, voids, status changes, exports, refused or failed requests), `CRITICAL` (permission changes, password resets, deleting or changing the status of an employee, server errors). |
| `statusCode` / `errorMessage` | The HTTP answer, and the error text for failures. |
| `ip` / `client` | Caller's IP (the real client IP behind Cloudflare) and the parsed browser / OS / device. |
| `requestId` | Also returned to the dashboard as the `X-Request-Id` response header. Show it in error toasts so a support ticket can quote it. |
| `durationMs` | How long the request took. |

Field names in `changes` are the database column names (`price`,
`isActive`, `fullName`…). Map the ones you care about to labels; show the rest
as they are. Money values come back as plain numbers in strings (`"85"`, not
`"85.000"`).

## 3. List: `GET /audit-logs`

Paginated like every list. Newest first by default. The rows are in `data` and
the paging in `meta`:

```json
{ "success": true, "data": [ …entries… ],
  "meta": { "page": 1, "limit": 20, "total": 134, "totalPages": 7 } }
```

| Query | Example | Notes |
| --- | --- | --- |
| `page`, `limit` | `1`, `20` | `limit` ≤ 100 |
| `dateFrom`, `dateTo` | `2026-11-01`, `2026-11-30` | Whole days, inclusive, in `timezone`. Omit for no bound. |
| `timezone` | `Asia/Amman` | Default `Asia/Amman` |
| `actorId` | employee id | |
| `actorRole` | `ADMIN` / `STAFF` / `INSTRUCTOR` | |
| `action` | `DELETE`, `FREEZE`… | |
| `module` | `subscriptions` | Values from `/filters` |
| `entityType`, `entityId` | `Member`, id | |
| `outcome` | `SUCCESS` / `FAILURE` | |
| `severity` | `INFO` / `WARNING` / `CRITICAL` | |
| `method` | `POST` / `PUT` / `PATCH` / `DELETE` / `GET` | |
| `statusCode` | `403` | |
| `ip` | `203.0.113.24` | exact |
| `search` | free text | employee name/email, summary (EN or AR), record name, record id, path, request id, IP |
| `sortBy` | `createdAt` (default), `durationMs`, `statusCode` | |
| `sortOrder` | `desc` (default) / `asc` | |

An impossible date (`2026-02-30`) or `dateFrom` after `dateTo` → `400`.

**Suggested table columns:** Time · Employee (name + role chip) · Summary
(`summaryAr`/`summaryEn`) · Area (`moduleLabel…`) · Severity chip · Outcome
(✓ / ✕ with status code) · IP / device · a "Details" button.

**Quick filter chips that work well:** "Failed sign-ins" (`action=LOGIN_FAILED`),
"Refused" (`statusCode=403`), "Critical" (`severity=CRITICAL`), "Deletes"
(`action=DELETE`), "Exports" (`action=EXPORT`).

## 4. Details: `GET /audit-logs/:id`

The full entry (§2). `404` if the id doesn't exist. Suggested layout:

1. Header: `summaryAr`/`summaryEn`, time, severity, outcome.
2. **Changes** table: Field · Before · After (from `changes`). For a delete,
   show `before` as "Deleted record". For a create, all "Before" cells are empty.
3. Who & where: employee, role, IP, `client`, `userAgent`.
4. Technical (collapsible): method, `route`, `path`, status, `durationMs`,
   `requestId`, `operation`, `requestBody`, `query`, `errorMessage`.

## 5. Actions

| `action` | When |
| --- | --- |
| `CREATE` / `UPDATE` / `DELETE` | Ordinary record writes |
| `EXPORT` | Any export / download |
| `LOGIN` / `LOGIN_FAILED` / `LOGOUT` | Sign-in and sign-out |
| `PASSWORD_CHANGE` | An employee changed their own password |
| `PASSWORD_RESET` | Someone reset another employee's password |
| `PERMISSION_CHANGE` | Permissions changed (`…/permissions`) |
| `STATUS_CHANGE` | Status changed (`…/status`) |
| Operations | The last word of the URL in capitals: `FREEZE`, `UNFREEZE`, `CANCEL_FREEZE`, `RENEW`, `UPGRADE`, `TRANSFER`, `VOID`, `ACTIVATE`, `DEACTIVATE`, `APPROVE`, `REJECT`, `CANCEL`, `COMPLETE`, `REOPEN`, `CHECK_IN`, `CHECK_OUT`, `CHECKOUT`, `SEND`, `IMPORT`, `CONFIRM`, `GENERATE`… |
| `VIEW` | Page loads, only if the server turns `AUDIT_LOG_READS` on |

Get the real list for the dropdown from `GET /audit-logs/filters` rather than
hard-coding it.

## 6. Statistics: `GET /audit-logs/stats`

Query: `dateFrom`, `dateTo` (default: the last 30 days up to today),
`timezone` (default `Asia/Amman`), optional `actorId`, `module`.

```json
{
  "range": { "dateFrom": "2026-11-17", "dateTo": "2026-11-23", "timezone": "Asia/Amman" },
  "cards": {
    "events": { "count": 312, "previous": 280, "changePct": 11.4 },
    "failures": 9,
    "denied": 4,
    "failedLogins": 3,
    "critical": 2,
    "activeEmployees": 7
  },
  "byAction":  [ { "action": "UPDATE", "count": 140 } ],
  "byModule":  [ { "module": "subscriptions", "labelEn": "Subscription", "labelAr": "اشتراك", "count": 88 } ],
  "topEmployees": [ { "actorId": "cm…", "name": "Sara", "role": "STAFF", "count": 120 } ],
  "perDay": [ { "date": "2026-11-23", "events": 41, "failures": 1 } ]
}
```

- `cards.events.previous` / `changePct`: the same-length period just before.
- `denied` = refused requests (401/403). `critical` = `severity: CRITICAL`.
- `perDay` has **one entry per day of the range**, zeros included: draw it
  directly.

## 7. Filters: `GET /audit-logs/filters`

```json
{
  "modules": [ { "key": "subscriptions", "labelEn": "Subscription", "labelAr": "اشتراك", "hasEntries": true } ],
  "actions": [ "ACTIVATE", "APPROVE", "CANCEL", "CREATE", "DELETE", "…" ],
  "actors":  [ { "id": "cm…", "name": "Sara", "role": "STAFF", "lastActivityAt": "2026-11-23T08:10:00.000Z" } ],
  "roles": ["ADMIN", "STAFF", "INSTRUCTOR"],
  "outcomes": ["SUCCESS", "FAILURE"],
  "severities": ["INFO", "WARNING", "CRITICAL"],
  "methods": ["POST", "PUT", "PATCH", "DELETE", "GET"]
}
```

`actors` = employees who appear in the log, most recently active first (up to
300). `hasEntries: false` areas can be greyed out.

## 8. A record's history: `GET /audit-logs/entity/:entityType/:entityId`

Same list format as §3, newest first, for one record. Use it for a "History"
tab on the member, subscription, package, invoice, employee… pages:

| Page | `entityType` |
| --- | --- |
| Member | `Member` |
| Employee (admin / staff / instructor) | `User` |
| Subscription | `Subscription` |
| Package | `Package` |
| Invoice | `Invoice` |
| Product | `Product` |
| Branch | `Branch` |
| Partnership | `Partnership` |
| Workout | `Workout` |
| Class | `ScheduleEvent` |
| Follow-up | `FollowUpProgram` |

(The full list is the `model` column in the backend's
`src/modules/audit-log/audit-catalog.ts`.) Sub-resources are recorded on their
parent: deleting a member's body record shows in the **member's** history with
`summaryEn` "… deleted Body records · member “…”".

## 9. My activity: `GET /audit-logs/me`

Same query and format as §3, always limited to the signed-in employee. No
permission needed: any employee can see what they did, including their own
refused attempts.

## 10. Export: `GET /audit-logs/export`

Same filters as §3 (no paging) plus `format=csv` (default) or `format=xlsx`.
Returns the file (`Content-Disposition: attachment; filename="audit-log-2026-11-23.csv"`),
not JSON. Up to the **50,000 newest** matching rows. Times are in the
requested `timezone`. Columns: Date & time, Employee, Email, Role, Action,
Area, Record type, Record, Record id, Summary, Changed fields, Outcome, Status,
Severity, Error, Method, Path, IP, Browser, OS, Duration (ms), Request id.

Download it as a blob (`responseType: 'blob'`). Exporting the audit log is
itself recorded.

## 11. Gotchas

- Rows appear a moment **after** the action (written once the response has
  been sent). After doing something, refetch with a short delay rather than
  expecting the new row instantly.
- List endpoints return rows in `data` and paging in `meta` (not
  `data.items`).
- `summaryAr` uses the employee's stored name, which may be in English.
- Failed actions read "… tried to delete package “X” (refused)". Use
  `outcome`/`statusCode` for colour, not the text.
- `changes` is empty for areas without a record snapshot (e.g. a report
  export or a checkout spanning several records). Show "No field changes".
- The log keeps everything; nothing expires it yet.
