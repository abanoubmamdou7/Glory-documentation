# 27 — Sandy round 3: AI budget, proactive messages, what-if, impact, coach copilot (2026-10-03)

> Audience: **web dashboard** (admin dashboard + instructor portal).
> Five screens/widgets. Same principle as before: Sandy thinks and proposes,
> people decide, and Sandy never deletes or changes business data.

## 1. AI usage & budget screen — `ai.usage.view` / `ai.budget.manage`

| UI | Endpoint |
| --- | --- |
| Header badge: OK / Background paused / All paused | `GET /ai-usage/budget` → `status.state`, `status.usedPct` |
| Cost dashboard (date range) | `GET /ai-usage/summary?from=&to=` |
| Budget form | `PATCH /ai-usage/budget` |

- Charts: `byDay` (line), `byFeature` (bar, colour by `tier`), `byActorType`
  (members vs staff vs system), tables `topEmployees` / `topMembers`.
- **Prices first.** Until `priceInputPer1M` + `priceOutputPer1M` are set,
  `pricesConfigured` is false and USD columns are 0 — show a "set your
  provider's prices" banner. Token caps work without prices.
- Explain the two thresholds next to the form: at 100 % of a limit the
  background jobs pause (daily brief, nutrition drafts); at limit ×
  `hardCapMultiplier` everything pauses (members see "temporary issue").
- Send `null` to remove a limit.

## 2. Proactive messages (Sandy → member) — `nudges.manage`

| UI | Endpoint |
| --- | --- |
| 4 template cards (status chip: Draft / Approved / Live) | `GET /nudges/templates` |
| Edit text (EN/AR) | `PATCH /nudges/templates/:trigger` |
| Approve button | `POST /nudges/templates/:trigger/approve` |
| Live toggle | `PATCH /nudges/templates/:trigger { enabled }` |
| Preview (pick a member) | `POST /nudges/templates/:trigger/preview { memberId }` |
| Settings (window, caps, thresholds) | `GET/PATCH /nudges/settings` |
| "Who would get a message now?" | `POST /nudges/run { dryRun: true }` |
| Sent log | `GET /nudges` |
| Results | `GET /nudges/stats?days=30` |

- Show `allowedPlaceholders` as insertable chips in the editor; a 400 names
  any unknown one.
- **Editing clears the approval and turns the template off** — warn before
  saving; then show Approve → Live as two steps.
- Results card per trigger: sent, replyPct, conversionPct ("came back",
  "renewed", "attended PT"). INBODY_IMPROVED has no conversion (engagement only).

## 3. What-if simulator (advisor screen) — `assistant.insights`

- Input box: `POST /admin-assistant/simulate { question }` → render
  `narrative` + a small table from `result.baseline` / `result.result` +
  `result.assumptions` (show them — they matter).
- `supported: false` → show the `message` and suggestion chips (price change,
  renewal target, discount, new members).
- Advanced form → `{ scenario: { type, … } }` (see docs/API.md).
- History: `GET /admin-assistant/simulations`.

## 4. Recommendation impact (decision board)

- On a recommendation marked DONE show "Impact will be measured on
  <decidedAt + 30 days>" and a **Measure now** button →
  `POST /admin-assistant/recommendations/:id/measure`.
- After measurement: `impactVerdict` chip (IMPROVED green, WORSENED red,
  NO_CHANGE grey, INCONCLUSIVE amber) + `impactSummary` + a before/after table
  from `impactValues.comparison`.
- Expect INCONCLUSIVE when measured right after DONE — Sandy refuses to credit
  an action with days-old data. That's intended.

## 5. Coach copilot (instructor portal) — INSTRUCTOR only

- Home card: `GET /coach-copilot/today` → `focus` as 3 lines (`messageAr` /
  `messageEn`), then sections for chats waiting, members who stopped coming,
  nutrition plans to review, PT today, renewals.
- In Coach Chat, a **"✨ Suggest a reply"** button →
  `POST /coach-copilot/conversations/:id/draft { instruction? }` → put `draft`
  **into the message input** (never auto-send). An optional small box lets the
  coach steer it ("shorter", "offer Thursday 6pm").
