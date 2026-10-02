# Freeze lifecycle — three different buttons, and a job that runs on its own

*Added 2026-09-30. Read this before wiring the freeze section of a
subscription screen: there are now **three** distinct actions on a frozen
sale, and picking the wrong one either takes money the member is owed or
quietly gives away days.*

---

## 1. The problem this closes

Freezing a subscription did two things: it set `status: FROZEN` and it pushed
`endDate` out by the frozen span. **Nothing ever set it back.**

So a member who froze for 14 days stayed `FROZEN` forever. And because the gym
gate only ever accepts an **`ACTIVE`** subscription, the turnstile kept
refusing them — *after* their freeze had ended — until a staff member noticed
and unfroze by hand. Members were being locked out of a gym they had paid for.

That is now fixed by a job that runs by itself. Nothing in the dashboard has to
call it.

---

## 2. The three actions

| Action | Endpoint | What it means | `endDate` | Quota | Timeline |
|---|---|---|---|---|---|
| **Freeze** | `POST /subscriptions/:id/freeze` | Pause the membership | pushed out by `days` | one freeze consumed | `FREEZE` row |
| **Unfreeze** | `POST /subscriptions/:id/unfreeze` | *The member came back* | pulled back by the **unused** days | counts the days actually used | `FREEZE` + `UNFREEZE` rows |
| **Cancel freeze** | `POST /subscriptions/:id/cancel-freeze` | *This freeze never happened* | back to **exactly** the pre-freeze value | **given back in full** | **nothing** — no FREEZE, no UNFREEZE |

> **Label them differently in the UI.** "Unfreeze" / "إلغاء التجميد" for both
> is how a receptionist erases a real 20-day freeze by accident. Suggested
> copy: **"Member returned — end the freeze"** vs
> **"Undo — this freeze was recorded by mistake"**, and put the undo behind a
> confirmation that says what it restores.

### Unfreeze — the member came back early
Body: `{ reason (required, ≥3 chars), endedAt? }`.

Freeze 14 days, member walks in on day 10 → `endDate` ends up **10** days
later than it started, not 14. The freeze stays on the timeline with
`actualDays: 10`, and the next freeze is measured against 10 days of the
package's allowance, not 14.

### Cancel freeze — undo a mistake
Body: `{ reason (required, ≥3 chars), refundFee? }`.

- `endDate` goes back by the **whole** planned span.
- The freeze row is **deleted**, so `freezeMaxCount` / `freezeMaxDuration` are
  given back — the member can be frozen again immediately.
- Therefore **`GET /subscriptions/:id/history` shows nothing about it**. Don't
  build a "cancelled freezes" list — there is no row to list. The trace is the
  sale's `notes`, which gains
  `Freeze cancelled (14 day(s) restored): <reason>`.
- The response adds
  `cancelledFreeze: { plannedDays, fee: { invoiceId, status, refundedAmount } | null }`.
- A sale whose own `endDate` had already passed while it sat frozen comes back
  **`EXPIRED`**, not `ACTIVE`.
- `409` when the sale is not `FROZEN`, or has no open freeze (including a
  second click on the same button).

#### The freeze fee — the one place the form needs two steps
If the package charged for the freeze, an invoice was issued for it.

| Fee invoice | Sending `cancel-freeze` | Result |
|---|---|---|
| Nothing paid | plain body | invoice → `CANCELLED`, no money moves |
| Something received | plain body | **`409`** naming the amount |
| Something received | `refundFee: true` | invoice → `REFUNDED` + an `OUT` row on the accounting ledger |

The `409` is deliberate: an undo must not move money silently. Surface the
message (it names the amount), then re-send with `refundFee: true` once the
user confirms the refund. **The refused call writes nothing** — the sale is
still `FROZEN`, so retrying is safe.

⚠ Freezes recorded **before 2026-09-30** carry no link to their fee invoice
(the column did not exist), so cancelling one of those touches no invoice —
settle it by hand from the invoice screen.

---

## 3. The job that runs on its own

`POST /membership-lifecycle/resume-frozen` — and it also fires **every 15
minutes, plus once whenever the server boots**.

When a freeze window runs out, the sale goes back to `ACTIVE` by itself. What
the dashboard needs to know:

- **A frozen sale will change status underneath you.** Don't cache
  `status: FROZEN` across a session; re-read on focus.
- The freeze is closed through the same code path as a manual unfreeze, with
  `endedAt` = the freeze's own planned end. So `endDate` is **not** changed
  (nothing went unused) and the timeline gains an `UNFREEZE` row whose
  `reason` is `"Freeze period ended"` and whose **`createdByName` is `null`** —
  render that as *System* / *تلقائي*, not as a blank actor.
- A freeze that is **still running** is never touched.
- A sale whose own `endDate` passed while it was frozen becomes **`EXPIRED`**,
  not `ACTIVE`.

Calling the endpoint yourself is only for an immediate catch-up or a preview:
body `{ asOf?, dryRun? }`, returns
`{ resumed: { count, subscriptionIds }, expired: { … }, failed: { … } }`.
`dryRun: true` lists the same sales and writes nothing. Permission
`subscriptions.manage`.

> `POST /membership-lifecycle/reconcile-expired` is **still not scheduled**, on
> purpose — it deactivates members, and nobody has asked for that to happen
> unattended. Keep whatever manual trigger you have for it.

---

## 4. Gotchas

1. **Status never moves through `PATCH /subscriptions/:id`.** That endpoint is
   clerical only. Freeze / unfreeze / cancel-freeze / void are the only ways.
2. `reason` is **required** and must be ≥3 characters on both unfreeze and
   cancel-freeze — a blank reason would make the audit trail useless, so the
   API refuses it rather than storing nothing.
3. **Only one freeze is open at a time.** Both endpoints act on *the* open
   freeze; there is no freeze id to pass.
4. The quota numbers live on the **package** (`freezeMaxCount`,
   `freezeMaxDuration`, `freeFreezesCount`, `freezePrice`) — read them from the
   package to show "2 of 3 freezes used" before the user even opens the dialog.
5. After a cancel-freeze the freeze count really does go down. If you cached
   "quota exhausted", refresh it.
