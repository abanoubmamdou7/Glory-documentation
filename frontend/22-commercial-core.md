# Commercial core — households · multi-beneficiary checkout · outstanding-balance policy · manager override

**Delivered 2026-09-24.** Four sales operating rules the gym asked for, all of
them moved **out of the browser and onto the server**. Nothing existing was
replaced: `POST /subscriptions`, `POST /payments` and the QR check-in behave
exactly as before unless a household, a checkout invoice or a stored policy is
involved.

| The rule | Where it lives now |
| --- | --- |
| One customer, several subscriptions, **one invoice** | `POST /sales/checkout` |
| A parent and a child may **share one phone number** | `POST /families` + `guardianId` on `POST /members` |
| The **system** decides what happens when a member owes money | `GET/PATCH /settings/check-in-outstanding`, enforced on every check-in |
| A manager override is **real, permissioned and recorded** | `checkins.outstanding.override` + `override`/`reason` on `POST /checkins` |

Full request/response reference: `docs/API.md` → *Families & guardians*,
*Checkout — one invoice, many beneficiaries*, *Outstanding-balance check-in
policy*.

---

## 1. Selling to a whole family in one go

```
POST /sales/checkout          (subscriptions.manage)
Idempotency-Key: <any string ≥ 8 chars, generated per checkout attempt>

{
  "payerMemberId": "<the mother>",
  "dueDate": "2026-10-24",
  "lines": [
    { "memberId": "<the mother>",   "packageId": "<annual gym>" },
    { "memberId": "<the daughter>", "packageId": "<swimming>" },
    { "memberId": "<the mother>",   "packageId": "<PT 10>", "discount": "10.000" }
  ],
  "payments": [ { "amount": "200.000", "method": "CASH" } ]
}
```

→ `201 { invoice, subscriptions[], idempotent }`.

Things to build against:

- **Generate the `Idempotency-Key` in the UI**, once per checkout attempt, and
  reuse it on retry (a network timeout, a double-clicked *Confirm*). Replaying
  it returns the original sale with `idempotent: true` instead of billing
  twice. Missing or shorter than 8 characters → **400**.
- **At least 2 lines.** A single subscription still goes to `POST
  /subscriptions` — that endpoint is untouched, so do not migrate existing
  one-line flows onto checkout.
- **Preview with `POST /sales/quote` per line.** The checkout total is the sum
  of the per-line quotes to the last fils — both use the same pricing code.
  ⚠ `taxRate` in a quote body is ignored; the package's own rate is
  authoritative.
- **One currency per checkout** (this backend performs no conversion
  anywhere). Mixed currencies → 400.
- **Each invoice line names its beneficiary** (`memberId`, `memberName`,
  `packageId`, `subscriptionId`) — print "Swimming — Sara (daughter)", not just
  "Swimming".
- **Partial payment is normal here.** A deposit leaves the invoice
  `PARTIALLY_PAID` (new status, below) with `balanceDue` and `dueDate` set.
  Take the rest later with the ordinary `POST /payments`.
- **All or nothing.** If one line fails (inactive package, overlapping
  subscription → 409) nothing at all is written. Show the error and let them
  fix that line; there is no half-finished sale to clean up.

> ⚠ For a checkout sale, **the invoice is the source of truth for money
> received.** A payment covering three subscriptions is not split across them,
> so do not read a checkout member's `subscription.receivedAmount` as "what
> they paid".

## 2. `PARTIALLY_PAID` — a new invoice status

`InvoiceStatus` is now `PENDING · PARTIALLY_PAID · PAID · CANCELLED ·
REFUNDED`. Anywhere the UI filtered or coloured on `PENDING` to mean **"still
owed"**, it must now match **`PENDING` or `PARTIALLY_PAID`**. Every report, the
member profile and the admin dashboard were updated the same way on the server.

Also: amending (`PATCH /invoices/:id`) a `PARTIALLY_PAID` invoice is now a
**409** — money has already been taken against it. Issue a refund/adjustment
invoice instead. (Before this change a partly-paid invoice still read as
`PENDING` and could be re-priced underneath a payment.)

## 3. Households and the shared phone number

A household grants exactly two things: a shared contact number, and one member
paying for another on one invoice. It does **not** pool memberships or gym
access — a dependent is a full member with their own `memberCode` and their own
subscription.

```
POST /families        { "name": "Amani household", "primaryMemberId": "<parent>" }
POST /members         { "fullName": "Sara", "phone": "<same as parent>",
                        "dateOfBirth": "2014-01-01", "guardianId": "<parent>" }
```

- **`guardianId` alone is usually enough** — the guardian's household is
  inherited, so you rarely send `familyId`. `familyRole` defaults to
  `DEPENDENT`.
- **Under 16 requires a guardian.** Creating a member whose `dateOfBirth`
  makes them younger than 16 without `guardianId` → **400**.
- **A duplicate phone is still a 409 for everyone else.** The number may only
  be reused by a member joining the household that already holds it. The 409
  message says so — surface it as *"this number belongs to another household"*,
  not *"invalid phone"*.
- Every member payload now carries `familyId`, `familyRole`, `guardianId` /
  `guardianName` (and `dependents` on the detail), so a dependent can be
  labelled at the desk without a second call.

> **Mobile login is unaffected.** At most one member per phone number may hold
> an app password, so logging in with a phone number is still unambiguous. Give
> the dependent their **own email** if they need their own app account.

> There is no `DELETE /families/:id` yet — deleting a household would strand
> several members on the same phone number at once, which the database
> (correctly) refuses. Ask for it if the dashboard needs it and it will be built
> with the phone-clearing step included.

## 4. Who gets in when they owe money — the gym decides, once

`GET /settings/check-in-outstanding` → `{ outstandingCheckInPolicy, graceDays,
scope, appliesToBranchId, … }`. `PATCH` the same path to change it (omit
`branchId` for the gym-wide default, pass one for a branch override — a branch
rule outranks the gym rule, and `scope` tells you which you are looking at so
the UI can show *inherited*).

| Policy | What the desk sees |
| --- | --- |
| `ALLOW_WARN` (**default**) | check-in succeeds, `warnings: ["Member owes 96.400 — settle before the next visit"]` |
| `GRACE` | succeeds until `dueDate + graceDays`, with the deadline in the warning; **403** after |
| `BLOCK` | **403** until settled |
| `MANAGER_OVERRIDE` | **403** unless a manager approves it |

Enforced on **all three** ways in: `POST /checkin/scan` (QR), `POST /checkins`
(manual/desk) and `POST /mobile/bookings/:id/check-in` (the member's own app) —
so the app cannot be used to slip past a block. Only the **desk** endpoint can
override; the QR and app paths return a 403 that sends the member to the desk.

The balance counted is every open (`PENDING`/`PARTIALLY_PAID`) invoice of
theirs **plus** open invoices that merely carry a line for them — so a child is
gated by the balance on the parent's checkout invoice that bought their
membership.

## 5. The manager override

```
POST /checkins   { "memberId": "...", "override": true,
                   "reason": "Manager approved the remaining balance at the desk" }
```

- `override: true` is **intent, not authority**. Without the
  `checkins.outstanding.override` permission (label **"Check In Members Who Owe
  Money"**, ADMIN always has it) the request is still a **403**.
- Every accepted override writes a `CheckInOverrideAudit` row: the member, the
  check-in, the exact amount outstanding at that moment, the policy in force,
  the reason typed, the branch, and who requested/approved it.
- Build the desk UI as: try the check-in → on 403, if the message offers an
  override *and* the user holds the key, show a **reason box** (required, ≤500
  chars) and retry with `override: true`. The two 403 messages are distinct on
  purpose:
  - *"Member owes 96.400. Resend with override:true and a reason to let them
    in."* → this user may override.
  - *"A manager must approve this check-in."* → they may not; fetch a manager.
- Successful check-ins now also return `outstandingBalance`,
  `outstandingPolicy` and `overrideApplied`, so the desk can show the balance
  on the confirmation without another call.

## Gotchas checklist

- [ ] `Idempotency-Key` generated per checkout attempt and reused on retry.
- [ ] `PENDING` filters widened to `PENDING, PARTIALLY_PAID`.
- [ ] Duplicate-phone 409 shown as a *household* problem, not a validation one.
- [ ] Check-in 403 handled on all three paths, with the reason box only where
      the user actually holds the override key.
- [ ] `checkins.outstanding.override` granted to whoever should hold it — it is
      seeded by the deploy but granted to nobody (ADMIN excepted).
- [ ] Don't read `receivedAmount` per subscription for checkout sales.
