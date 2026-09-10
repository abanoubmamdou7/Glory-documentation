# 20 — Reports & Analytics (the 14 report screens)

> Every card in the Figma "Reports & Analytics" grid is now backed by its
> own **cards / list / export** trio under `/reports/<name>/…` (the older
> `GET /reports/<name>` endpoints still return the aggregate objects used by
> the Admin Dashboard). Full field reference: `docs/API.md` → "Reports &
> Analytics — row-level reports". Postman:
> `postman/GloryGym-ReportsAnalytics.postman_collection.json`.
> Standalone hand-off reference with request/response examples for every
> endpoint: `docs/REPORTS_ANALYTICS_API.md`.

## 1. The pattern (same on every screen)

| Piece | Endpoint | Notes |
| --- | --- | --- |
| Stat tiles | `GET /reports/<name>/cards` | Same filters as the list (minus page/limit). |
| Table | `GET /reports/<name>/list` | `page`, `limit`, `search`, `sortBy`, `sortOrder` + the screen's filters. Response is the usual `{ data: rows[], meta }`. |
| Export button | `GET /reports/<name>/export` | CSV (UTF-8 BOM), same filters, no pagination, max 5000 rows. Download as a blob. |

- **Dates:** `dateFrom` / `dateTo` (`YYYY-MM-DD`, inclusive). Omitted =
  first day of the current month → today. Send the gym's timezone
  (`timezone=Asia/Amman` or leave the `Africa/Cairo` default) consistently.
- **Money:** strings with 3 decimals (`"2200.000"`). Format client-side.
- **Permissions:** `reports.view` for all report screens, **except**
  Payroll / Salary / Month Target which are `payroll.read` (salary writes
  `payroll.salary.manage`), and the Staff attendance *writes* which are
  `staff.manage`. ADMIN bypasses.

## 2. Screen → endpoint map

| Figma screen | Endpoints | Filters (query) | Notes |
| --- | --- | --- | --- |
| **Sales** | `/reports/sales/{cards,list,export}` | date, `createdById`, `assignedSaleId`, `transactionType` (Transaction Type), `source` (NEW/RENEWAL/UPGRADE), `salesType` (SUBSCRIPTION/PRODUCTS = "Sales Type"), `status`, `branchId`, `packageId`, `search` | Rows mix package sales and product sales (`category`). Cards: total vs "Filtered" = without / with the non-date filters. |
| **Members** (Members tab) | `/reports/members/{cards,list,export}` | date (joining date), `status`, `gender`, `relationType`, `createdById`, `branchId`, `membershipStatus`, `includeDeleted`, `search` | 11 cards. `mobile` is already the joined `+country phone`. Action icons → `GET /members/:id` / `PATCH /members/:id`. |
| **Members** (Memberships tab) | `/reports/memberships/{cards,list,export}` | date, `packageId`, `membershipType` ("Package List Filter"), `memberStatus`, `status`, `branchId`, `search` | "Check-Ins 0 / 150 Day" = `checkIns.used` / `checkIns.allowance` + `checkIns.unit` (DAYS or SESSIONS). |
| **Revenue** | `GET /reports/revenue` (existing aggregate) | date, `branchId` | Unchanged. |
| **Booking** | `/reports/booking/{list,export}` | date, `packageId` (= "Service"), `serviceType`, `instructorId`, `branchId`, `search` | One row per instructor × service × type; `totalMembers` is distinct members. |
| **Account Balance** | `/reports/account-balance/{list,export}` | `onlyOutstanding`, `branchId`, `search`, optional date | Per member. Action icon → `GET /invoices?memberId=`. |
| **Staff** | `/reports/staff/{list,export}` (alias of `/staff-attendance`) | date, `userId`, `role`, `branchId`, `source`, `openOnly`, `search` | Rows are real clock-in/out days. Employees create them with `POST /staff-attendance/check-in` / `check-out`; HR with `POST /staff-attendance`. |
| **Transaction** | `/reports/transactions/{cards,list,export}` | date, `transactionType`, `direction`, `category`, `kind`, `memberId`, `subscriptionId`, `branchId`, `createdById`, `search` | `outcome` is negative. Second "Transaction Type" column = `categoryLabel`. Manual rows: `POST /transactions`. |
| **Canceled Memberships** | `/reports/canceled-memberships/{list,export}` | date (cancel date), `type` (membership type), `packageId`, `branchId`, `canceledById`, `search` ("Search Canceled By" or member) | `voidedAmount`, `refundedAmount`, `canceledBy`, `canceledAt`, `reason`. |
| **Accounting Entry** | `/reports/accounting-entry/{cards,list,export}` | date, `groupBy=SERVICE` (Members tab) / `PACKAGE` (Memberships tab), `createdById`, `assignedSaleId`, `transactionType`, `source`, `salesType`, `branchId` | 9 cards; table = one line per service (Subscriptions / Pt / Jf / Retail / Other) with the 12 measures. |
| **Follow-up Program Tracking** | `GET /follow-up-programs` + `/export` | date (expected date), `packageId`, `gender`, `createdById`, `instructorId`, `recordType`, `status`, `search` | Existing list, now with `delayDays`, `sentEmails.count`, `createdBy`. AI Evaluation / AI Feedback come back as stored (null until a generator exists). |
| **Payroll** | `/reports/payroll/{list,export}` | date, `payrateType`, `role`, `userId`, `search` | `userBalance = earnedAmount + deferredAmount` (definitions in API.md). |
| **Salary** + "Add Salary Details" | `/reports/salary/{list,export}` (alias of `/salaries`), form: `GET /salaries/preview`, `POST /salaries`, `PATCH`/`DELETE /salaries/:id` | `period` (YYYY-MM), `userId`, `role`, date, `search` | See §3. |
| **Month Target (JOD)** | `/reports/month-target/{list,export}` | `period`, `type` (INSTRUCTOR/STAFF), `userId`, `onlyWithTarget`, `search` | Second table (sales per person) = `GET /reports/sales/list?employeeId=<id>&dateFrom&dateTo`. "Add Salary Details" button → §3. |
| **Assessment Evaluation** | `/reports/assessments/{cards,list,questions,export}` | date (send date), `instructorId`, `memberId`, `type` (GOOD/BAD), `search` | Call `questions` once to build the star columns; each row's `answers[]` is keyed by `questionId`. |

## 3. "Add Salary Details" form wiring

1. Instructor select → `GET /instructors?limit=100` (+ `/staff` if staff can
   be paid).
2. On select (and on every change of Month Target / Insurance / Discounts /
   Salary Increase): `GET /salaries/preview?userId&period&monthTarget&insurance&discounts&salaryIncrease`
   → fill the read-only **Base Salary**, **Remaining Target**, **Net Salary**.
   `400` "Set the employee's base salary in Team Management first" means
   the profile has no `baseSalary` — link to the employee's Job Info tab.
3. "Month Salary" month picker → `period` (`YYYY-MM`). The numeric "Month
   Target" box → `monthTargetCount`; the JOD one → `monthTarget`.
4. Save → `POST /salaries`. `409` = that month already has an entry (open it
   and `PATCH` instead).

Maths the backend applies: monthSalary = base + increase; total = month −
insurance − discounts; net = total; remainingTarget = monthTarget − revenue
attributed to the employee in that month (negative = exceeded).

## 4. Staff attendance (the Staff report's data source)

- Employee dashboard: a **Check in** button → `POST /staff-attendance/check-in`
  (409 if already in today), **Check out** → `POST /staff-attendance/check-out`
  (404 no check-in today, 409 already out). `GET /staff-attendance/me` for
  the employee's own history.
- HR: manual day entry `POST /staff-attendance { userId, date, checkInAt,
  checkOutAt? }`, edit `PATCH /staff-attendance/:id`, delete.
- One row per employee per calendar day; `hours` is null while the day is
  still open.

## 5. Gotchas

- **Run the ledger backfill once** after this deploy
  (`POST /transactions/backfill`, `payments.manage`) or the Transaction
  report starts empty for history. Safe to repeat.
- **Product sales are new**: the "products" rows on Sales / the Retail line
  on Accounting Entry only appear once the front desk sells products via
  `POST /product-sales` (see API.md → Product sales). Until then they are
  simply zero, not broken.
- **Joining fees** now exist on the sale form: send `joiningFees` on
  `POST /subscriptions`; the invoice gets a second line and the Sales /
  Accounting reports show it separately.
- **`outcome` on the Transaction cards is negative** ("-6,345.425") — render
  as-is; `total = income + outcome`.
- **Two "Transaction Type" columns** on the Figma Transaction table: the
  first is the payment method (`paymentMethod`), the second is
  `categoryLabel` ("Card Processing Fees", "Purchase", …).
- **Check-ins per membership** only count visits recorded after this
  deploy (older check-ins have no subscription link), and `remainingSessions`
  is still not decremented by visits.
- **Members cards default to all-time**, but the members *list* range
  applies to the joining date (default current month) — pass a wide
  `dateFrom` for an "all members" table.
- **Payroll / Month Target definitions are inferences** from column names
  (documented in API.md); tell backend if the gym computes them
  differently.
- CSV exports start with a UTF-8 BOM so Excel shows Arabic correctly; if you
  read the text in JS the BOM is invisible (`fetch().text()` strips it).
