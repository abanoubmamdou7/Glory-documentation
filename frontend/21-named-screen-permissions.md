# Named screen permissions (2026-09-19)

Closes the "Glory Gym KEYS — missing named-screen permissions" collection.
Every sidebar screen now has its own permission key, so an employee can be
given exactly the screens they need instead of a module-wide blanket key.

**The blocker this removes:** the wizard used to strip these strings and
`PUT /{admins|staff|instructors}/:id/permissions` answered
`400 permissionKeys contains unknown keys: …`. It now returns `200`.

---

## 1. How a named key is shaped

The catalog is code-owned (one key = one thing a guard actually checks), so
each named screen is a dotted key whose **label is the exact string you
send**:

```
{ "key": "settings.company.view", "label": "View Company Data", "group": "Settings" }
```

`PUT …/permissions` accepts **either side** (label match is
case-insensitive). What gets stored and returned is always the key:

```jsonc
// PUT /staff/{id}/permissions
{ "grantAll": false, "permissionKeys": ["View Company Data", "reports.sales.view"] }

// 200
{
  "id": "…",
  "grantAll": false,
  "permissionKeys":   ["settings.company.view", "reports.sales.view"],
  "permissionLabels": ["View Company Data", "View Sales Report"],
  "permissions":      ["settings.company.view", "reports.sales.view"], // legacy alias
  "chatScope": null
}
```

**`permissionLabels` is new** and is also on `GET /auth/me` and
`GET /{role}/:id/permissions`. If your RBAC matches on the named strings,
read `permissionLabels`; if it matches on dotted keys, read `permissions`.
You no longer need to build a key→label map from `GET /permissions`.

> There is **no `POST /permissions`**. The catalog ships in code and is
> published idempotently by `npm run prisma:seed`, which already runs on
> every deploy. `GET /permissions` is the source of truth for the wizard.

---

## 2. Three rules that make these safe to roll out

1. **Additive, never a replacement.** A named key opens that screen's READ
   endpoints *alongside* the module's legacy key. `GET /company-pages`
   accepts `settings.company.view` **or** `settings.view`. Nobody who can
   reach a screen today loses it when you start granting named keys.
2. **"View" means view.** Saving on a screen still requires that module's
   `.manage` key. `View Company Data` alone → `GET /company-pages` `200`,
   `PATCH /company-pages/vision` `403`.
3. **Siblings are isolated.** `View Sales Report` does not open
   `/reports/members/list` — that needs `View Members Report`.

---

## 3. Already-live strings (not duplicated)

These were already in the catalog under that exact label, so they resolve
to the existing key. Send them as-is:

| String | Resolves to |
|---|---|
| `View Dashboard` | `dashboard.view` |
| `View Members` | `members.read` |
| `View Subscriptions` | `subscriptions.read` |
| `View Packages` | `packages.read` |
| `View Organization Hierarchy` | `organization.view` |
| `View Own / Team / All Attendance`, `Manage Attendance` | `attendance.*` |

---

## 4. Group names — three deltas from the collection

Group strings are display-only, and we reused an existing group rather than
adding a near-duplicate. If you render section headings from the API, use
these:

| Collection said | API returns |
|---|---|
| `Chat` | `Coach Chat` |
| `Scheduling & Classes` | `Reservations & Scheduling` |
| `Staff and Instructors` | `Staff And Instructors` (capital A) |

New groups added as requested: **Members Management**, **Plans &
Memberships**, **Invoice**.

---

## 5. What is NOT enforced server-side (and why)

Publishing a key and enforcing it are separate steps on purpose — enforcing
a key nobody holds yet logs everyone out of that screen on deploy.

**Open to any logged-in employee today (key published + assignable, endpoint
still open):** Branches, Products, Departments & Facilities, Videos Library,
Follow-up Programs, Push Notifications. Gate these in the UI from
`permissionLabels`; say the word when the keys are rolled out and the server
side is one line per endpoint.

**Scope-resolved in the service (the key controls the nav item, not the
rows):** `View Overview Target` and `View Monthly Closing Target`
(`/performance/*`, `/targets/plans/:id/close`), `View Staff Chats`
(`/staff-chat/*` clamps to the caller's own chat scope), `View Staff
Attendance` (`attendance.view.own/.team/.all` still decides which rows come
back).

Everything else in the collection is enforced: Settings (5 screens),
Check-in, Partnership, Invoice/Ledger/Sales Approvals, Product Sales, Sandy
AI, Admins/Instructors directories, Salary Details, all 14 report cards and
all Targets tabs.

---

## 6. Gotchas

- **`grantAll` flips to `false` for anyone who had "select all".** It is
  computed as *granted count == catalog size*, and the catalog went 71 →
  119. Their existing 71 keys still work (every gate accepts the legacy
  key), but re-tick "select all" to pick up the new 48.
- **`View Partnership` (singular) is a new key**; the legacy
  `View Partnerships` (plural, `partnerships.read`) still opens the same
  screen. Both appear in the wizard until you retire the legacy one.
- **`View Members` is ambiguous in the catalog** — `dashboard.view.members`
  carries the same label. The alias resolves it to `members.read` (the
  Members screen), which is what you want; send the dotted key if you
  specifically mean the dashboard card.
- **`View Sandy AI`** opens the Sandy knowledge screens. The Admin
  Assistant chat on the same page is gated separately by `assistant.use`
  (see `16a-admin-assistant-permissions.md`).
- **`View Sales Approvals Screen`** also lets an *approver* list the queue:
  `GET /sale-change-requests` now accepts `sales.changes.approve` too
  (previously it needed `sales.changes.request`, which approvers may not
  hold). Raising a request still needs `sales.changes.request`.
