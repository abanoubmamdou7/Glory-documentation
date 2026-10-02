# 26 — Sandy as advisor + the coach's nutrition-plan review (2026-10-03)

> Audience: **web dashboard** (admin dashboard + instructor portal).
> Two new screens. Both follow the same principle: **Sandy thinks and proposes,
> a human decides, and Sandy can never delete or change business data.**

---

## 1. Advisor screen (admin dashboard) — `assistant.insights`

Sandy reads the gym's real numbers and gives management a financial analysis,
a 3-month outlook and concrete recommendations.

| Screen part | Endpoint |
| --- | --- |
| "Generate analysis" button (+ optional focus question, months 3–12, branch) | `POST /admin-assistant/insights` |
| Report history list | `GET /admin-assistant/insights` |
| Report page | `GET /admin-assistant/insights/:id` |
| Numbers-only view (charts, no AI) | `GET /admin-assistant/insights/metrics` |
| Decision board | `GET /admin-assistant/recommendations?status=` |
| Accept / Dismiss / Done buttons on a recommendation | `PATCH /admin-assistant/recommendations/:id` |

**Report page layout suggestion:** `analysis.headline` big → `analysis.summary`
→ findings as cards coloured by `severity` (`POSITIVE` green / `WATCH` amber /
`RISK` red) → the forecast (`snapshot.metrics['forecast.collected.next3Months']`
as a chart + `analysis.forecast.commentary`) → recommendations ordered by
`priority`, each with **Accept / Dismiss** and a note box → `risks` and
`questionsForManagement`.

**Show the evidence.** Every finding/recommendation has `evidence` = metric
keys from `snapshot.metrics`. Render them as small chips that reveal the
number — it's what makes the advice trustworthy, and it is guaranteed real
(ungrounded items are dropped server-side).

**Gotchas**
- Generation takes **~20–40 s** — show a progress state; it's a single
  awaitable request. `503` = AI provider down / unusable reply → "try again".
- Accepting a recommendation **does not do anything in the system** — it's a
  decision log. The note matters: Sandy reads past decisions and stops
  re-proposing what management dismissed. Ask for a reason on Dismiss.
- If `snapshot.metrics['dataQuality.paymentsMissingFromLedger'] > 0`, show a
  banner: the accounting ledger history isn't backfilled (`POST
  /transactions/backfill`). Revenue numbers in the advisor are still correct.
- Money is JOD; no currency conversion (project rule).

### Daily brief (home widget) — added round 2
Every morning (07:00 Amman by default) Sandy writes a brief automatically.
- Home card: `GET /admin-assistant/insights/daily/latest` → show
  `analysis.greeting`, `headline`, `yesterday`, the 3 `todayPriorities`.
- "Refresh / write today's" button: `POST /admin-assistant/insights/daily/run`
  (returns the existing one if already written today).
- **People** section: `analysis.peopleHighlights` — PRAISE (green, can be
  shared with the person), SUPPORT / WATCH (manager-only; label them so).
- **Today's call list**: render from `snapshot.metrics['actions.callList']`
  (name, member code, coach, reasons — map `NO_VISIT_30_DAYS`,
  `EXPIRES_<date>_<package>`, `OWES_<amount>_JOD_OVERDUE` to friendly chips;
  link the member code to the profile) + `analysis.callListAdvice` above it.
- Forecast: show `analysis.forecast.confidence` as a badge and, when present,
  `snapshot.metrics['forecast.track']` ("how accurate Sandy has been").
- "Ask Sandy about this" box: `POST /admin-assistant/insights/:id/ask`
  `{ question }` → `{ answer }`; history is in the report's `followUps`.
- Coach/staff tables for a "Team" tab: `snapshot.metrics['people.coaches']`,
  `['people.sales']`, `['people.attendance']`.

### Chat changes (`POST /admin-assistant/chat`)
- New `error: "WRITE_REFUSED"` — the user asked it to delete/change/add data.
  It's a normal `200` with a friendly `answer` explaining it's read-only. Render
  like `OUT_OF_SCOPE`, not as a red error.
- Answers now often end with a "💡" suggestion line. Nothing to change in the UI.

---

## 2. Nutrition review queue (instructor portal + admin)

After every InBody test, Sandy drafts a nutrition plan for the member and
sends it to **their coach**. The member sees nothing until the coach acts.

| Screen part | Endpoint |
| --- | --- |
| Sidebar badge | `GET /nutrition-plans/pending-count` |
| Queue (tabs: Pending / Approved / Rejected) | `GET /nutrition-plans?status=` |
| Plan review page | `GET /nutrition-plans/:id` |
| **Approve** | `POST /nutrition-plans/:id/approve` `{ coachNote? }` |
| **Edit & approve** | `POST /nutrition-plans/:id/approve` `{ content, coachNote? }` |
| **Reject** (reason required) | `POST /nutrition-plans/:id/reject` `{ reason }` |
| Manager: reassign coach | `PATCH /nutrition-plans/:id/reviewer` |
| Manager / coach: "generate now" for a test | `POST /nutrition-plans/generate` `{ bodyRecordId }` |

**Who sees what:** a coach automatically sees only plans they review — no
permission needed. `nutrition.manage` (or ADMIN) sees everything, can reassign
and regenerate. Show "Reassign" only with that key.

**Review page layout suggestion:** left = the InBody comparison
(`comparison.current` vs `comparison.previous`, colour the `deltas`), right =
the plan (`draftContent`): calories + macros, meals, guidelines, avoid. Show
`draftContent.reviewerNotes` in a "Sandy's reasoning (only you see this)" box.

**Edit & approve:** pre-fill an editor with `draftContent` and send the **whole
edited object** as `content`. The server checks the same safety rules it
applies to Sandy — a `400` message lists exactly what's wrong (e.g. calories
below the member's BMR, macros not adding up to the calories). Show it inline.
Sandy's original stays in `draftContent`; the member gets `finalContent`.

**Statuses:** `PENDING_REVIEW` (act on it), `APPROVED` (delivered — show
`deliveredAt`, `editedByCoach`), `REJECTED` (final for that test), `FAILED`
(Sandy couldn't produce a safe plan — `lastError`; managers/coach can press
"generate now"), `GENERATING` (in progress).

**Gotchas**
- A rejection is final for that InBody test; the next test gets a new plan.
- `409` on approve/reject = someone already decided it — refresh.
- Only tests **taken in the last 48 h** are auto-planned; for an older test use
  "generate now".
