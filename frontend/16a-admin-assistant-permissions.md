# 16a — Admin Assistant: permission-aware access (update 2026-09-12)

> Audience: **web dashboard frontend** (and whoever assigns staff permissions).
> This documents a change to the Admin Assistant ([16-admin-assistant.md](16-admin-assistant.md)),
> not a new feature. If you already built the chat UI, the API shape is
> unchanged — but who can use it, and what each user can ask about, changed.

## TL;DR

- The Admin Assistant is **no longer ADMIN-only**. It is now gated by a new
  permission, **`assistant.use`** (ADMIN still has full access automatically).
- It is now **permission-aware**: which database tables a caller can query is
  decided by their **other** permissions. An **accountant** can ask about
  money/billing/finance but not staff/HR; an **HR** user can ask about staff but
  not revenue/salaries. Cross-area questions are **refused**.
- **No endpoint paths or request/response shapes changed.** The only new things
  you may want to handle in the UI: gate the nav on `assistant.use`, and treat a
  new `error` value — **`PERMISSION_SCOPE`** — like the existing `OUT_OF_SCOPE`
  (a friendly refusal, still HTTP `200`).
- **Security fix bundled in:** members' private medical records
  (`MemberMedicalRecord`) are now blocked from the assistant for **everyone,
  including ADMIN**.

---

## Why

The request: *"make the assistant work by permissions — if an accountant gives
it a command that belongs to HR, refuse, and vice-versa."* So instead of one
all-seeing ADMIN assistant, each employee gets an assistant scoped to the data
their role already lets them touch.

---

## 1. Access is now `assistant.use` (not the ADMIN role)

| Before | After |
| --- | --- |
| `@Roles(ADMIN)` — only ADMINs could open it; everyone else `403`. | Gated by the **`assistant.use`** permission. ADMIN bypasses (always allowed). Any STAFF/INSTRUCTOR **granted `assistant.use`** can open it. |

- Grant `assistant.use` from Team Management (`PUT /{admins|staff|instructors}/:id/permissions`), same as any permission — its label is **"Use the Admin AI Assistant"** (group **"Admin Assistant"**).
- A caller **without** `assistant.use` gets **`403`** on every `/admin-assistant/*` route. **Gate the nav item on `assistant.use`** (present in the `permissions` array from `GET /auth/me`), instead of on the ADMIN role.
- Existing staff have **no** access until you grant it (ADMIN unaffected).

---

## 2. What each user can ask about (the scope map)

A caller can only query the tables their permissions unlock. Everyone with
`assistant.use` can also read **reference/lookup** tables (branches, packages,
facilities, products, departments, specialities) so answers can show real names.
**ADMIN / grant-all users reach everything readable.**

| Business area (what they can ask about) | Unlocked by ANY of these permissions |
| --- | --- |
| **Members** (members, family, body records, assessments, onboarding answers) | `members.read`, `members.manage`, `dashboard.view.members`, `reports.view` |
| **Sales & Subscriptions** (subscriptions, freeze/transfer history, sale-change requests) | `subscriptions.read`, `subscriptions.manage`, `sales.changes.request`, `sales.changes.approve`, `dashboard.view.sales`, `reports.view` |
| **Partnerships** | `partnerships.read`, `partnerships.manage` |
| **Billing** (invoices, invoice items, payments) | `invoices.read`, `invoices.manage`, `payments.manage`, `dashboard.view.revenue`, `reports.view` |
| **Accounting** (the Transactions ledger) | `invoices.read`, `payments.manage`, `reports.view`, `payroll.read`, `dashboard.view.revenue` |
| **Payroll & Salaries** (salaries, payroll, month targets) | `payroll.read`, `payroll.manage`, `payroll.salary.manage`, `payroll.targets.manage` |
| **Targets & Performance** | `targets.read`, `targets.manage`, `targets.approve`, `performance.view.*`, `performance.export` |
| **Staff & HR** (users, managers, documents, shifts, sub-departments, attendance) | `staff.read`, `staff.manage`, `admins.read`/`.manage`, `instructors.read`/`.manage`, `staff.documents.*`, `organization.view`/`.manage`, `attendance.view.team`, `attendance.view.all`, `attendance.manage` |
| **Bookings & Check-ins** | `checkins.read`, `checkins.manage` |
| **Scheduling & Classes** | `schedule.read`, `schedule.manage`, `reservations.manage` |
| **Workouts** | `workouts.manage` |
| **Videos Library** | `videos.manage` |
| **Follow-up Programs** | `followup.manage` |
| **Coach Chat** | `chat.manage`, `chat.manage.department`/`.branch`/`.all` |
| **Sandy AI** (knowledge + member chats) | `sandy.read`, `sandy.manage` |
| **Push Notifications** | `notifications.manage` |
| **Content & Settings** (pages, FAQs, email templates, settings) | `settings.view`, `settings.manage` |
| **Reports & Exports** (export jobs) | `reports.view`, `performance.export` |

> Note `attendance.view.own` deliberately does **not** unlock the Staff & HR
> data — it's a "just me" scope, and the assistant has no row-level filtering, so
> granting it the whole staff-attendance table would over-share. Team/all/manage
> tiers do unlock it.

**So, worked examples:**

- **Accountant** = `assistant.use` + `invoices.read` + `payments.manage` + `reports.view`
  → can ask about **Billing, Accounting, Sales, Members** (business metrics) —
  **cannot** ask about staff, salaries or attendance.
- **HR** = `assistant.use` + `staff.read` + `attendance.view.all` + `organization.view`
  → can ask about **Staff & HR** — **cannot** ask about revenue, invoices or salaries.

---

## 3. The new refusal — `error: "PERMISSION_SCOPE"`

When a caller asks about an area they don't have permission for, the response is
still HTTP **`200`** (a product state, exactly like `OUT_OF_SCOPE`) with:

```json
{
  "success": true,
  "data": {
    "conversationId": "clx1...",
    "messageId": "clx2...",
    "answer": "That question concerns data you don't have permission to access, so I can't answer it. Based on your permissions I can help with: Billing (Invoices & Payments), Accounting (Transactions ledger), Members. If you need access to other data, ask your administrator to grant the permission.",
    "sql": null,
    "rowCount": null,
    "rows": null,
    "error": "PERMISSION_SCOPE",
    "meta": { "model": "gpt-5.6-luna", "latencyMs": 900 }
  }
}
```

- The `answer` already names the areas the caller *can* ask about (Arabic when
  they asked in Arabic). **Just render `answer`** — don't show a red error.
- Add `PERMISSION_SCOPE` next to `OUT_OF_SCOPE` in your "these are friendly
  refusals, not failures" branch. Everything else about `POST /chat` is
  unchanged.

---

## 4. `GET /admin-assistant/schema` is now per-caller

Same shape as before, but `tables` now reflects **the calling user's** scope, so
an accountant and an HR user get different lists. Use it to drive a
"you can ask about…" hint that actually matches the signed-in user. `blockedTables`
now also lists `MemberMedicalRecord`.

---

## 5. Security fix — medical records blocked for everyone

`MemberMedicalRecord` (members' private lab/X-ray summaries and file URLs,
app-private to the member + Sandy) is now a **blocked table** — the assistant
cannot query it for **any** caller, ADMIN included. A question that would need it
comes back as a refusal, and no SQL ever touches that table.

---

## What did NOT change

- Endpoint paths, request bodies, and response fields — identical.
- Conversation list / history / rename / delete — identical (still per-user).
- The `OUT_OF_SCOPE` / `UNSAFE_SQL` / `EXEC_ERROR` / `GENERATION_ERROR` /
  `PROVIDER_NOT_CONFIGURED` states — identical; `PERMISSION_SCOPE` is added
  alongside them.
- Everything in [16-admin-assistant.md](16-admin-assistant.md) about building the
  chat UI still applies.

---

## Backend / deploy notes (FYI)

- `assistant.use` is seeded by `prisma:seed` (runs on deploy). The catalog is now
  **71 keys**.
- One edge for whoever manages permissions: a **non-ADMIN** who previously held
  *all* 70 keys is now 70/71, so their computed `grantAll` flips to `false` and
  they become *scoped* until re-granted "select all" (which picks up the 71st).
  ADMINs are unaffected.
- Enforced in two layers server-side: the SQL-writing model only ever sees the
  caller's scoped tables, and the SQL guard hard-rejects any out-of-scope table —
  so scoping can't be talked around.

See [../../docs/API.md](../../docs/API.md) → "Admin Assistant" for the full
reference.
