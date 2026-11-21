# 07 — First-Login Onboarding (dynamic intake questionnaire)

> Audience: **mobile app**. Member Bearer token required. Covers the intake
> wizard shown **once, right after the member's first login**.
>
> ⚠ **This is now fully dashboard-configurable** — it replaced an earlier
> fixed 8-section/44-field form (see CLAUDE.md's progress log if you built
> against that older version). The question set, wording, order, and even
> which fields exist can change from Settings at any time with **no app
> release** — your screens must render the wizard generically from
> `GET /mobile/onboarding/questions`, not from hardcoded field names.

| # | Method & path | Purpose |
| --- | --- | --- |
| 1 | `GET /mobile/onboarding/status` | Show the wizard or go home? (call after every login) |
| 2 | `GET /mobile/onboarding/questions` | The active question list — build the wizard's screens from this |
| 3 | `POST /mobile/onboarding` | Submit all answers in one call (final step) |
| 4 | `GET /mobile/onboarding` | Read the saved answers back (404 if never submitted) |

## Flow

```mermaid
sequenceDiagram
  participant App
  participant API
  App->>API: POST /mobile/auth/login
  API-->>App: tokens + member (onboardingCompleted: false)
  App->>API: GET /mobile/onboarding/questions
  API-->>App: [ { id, questionEn, questionAr, type, options, required, sortOrder, showIfQuestionId, showIfValue } ]
  Note over App: render one screen/field per question, by `type`;<br/>hide/show conditional questions live as answers change
  Note over App: collect answers locally (no per-question API calls)
  App->>API: POST /mobile/onboarding {answers: [...]}
  API-->>App: 201 {completedAt, answers}
  Note over App: next logins: status → completed:true → straight to home
```

- **Shortcut:** the login response's `member` object (and
  `GET /mobile/auth/me`) carries **`onboardingCompleted: boolean`** — route
  straight from the login response without an extra call when `true`.
- The wizard state lives **client-side** — nothing is sent until the final
  step submits everything in one `POST`.
- Submission is **one-time**: a second `POST` → `409` ("already completed").
  There is no edit screen; treat the wizard as write-once.
- There is **no `prefill`** in this version — every question is
  independent of the member's account fields (name/phone/gender collected
  at signup are separate from whatever the admin chooses to ask here,
  even if it's worded similarly).

## 2. `GET /mobile/onboarding/questions` — build the form from this

```json
[
  { "id": "q_occupation", "questionEn": "What is your current occupation?",
    "questionAr": "ما هي مهنتك الحالية؟", "type": "TEXT",
    "options": null, "required": false, "sortOrder": 1,
    "showIfQuestionId": null, "showIfValue": null },

  { "id": "q_days", "questionEn": "How many days per week can you commit to training?",
    "questionAr": "كم يومًا في الأسبوع يمكنك الالتزام بالتدريب؟", "type": "NUMBER",
    "options": null, "required": true, "sortOrder": 2 },

  { "id": "q_injuries", "questionEn": "Do you have any injuries?",
    "questionAr": "هل لديك أي إصابات؟", "type": "BOOLEAN",
    "options": null, "required": true, "sortOrder": 3 },

  { "id": "q_injury_details", "questionEn": "Describe your injury and when it happened",
    "questionAr": "اذكر موضع الإصابة وتاريخها", "type": "TEXT",
    "options": null, "required": true, "sortOrder": 4,
    "showIfQuestionId": "q_injuries", "showIfValue": "true" },

  { "id": "q_time", "questionEn": "What is your preferred training time?",
    "questionAr": "ما هو الوقت المفضل للتدريب؟", "type": "SINGLE_CHOICE",
    "required": true, "sortOrder": 5,
    "options": [
      { "value": "morning", "labelEn": "Morning", "labelAr": "صباحًا" },
      { "value": "noon", "labelEn": "Noon", "labelAr": "ظهرًا" },
      { "value": "evening", "labelEn": "Evening", "labelAr": "مساءً" },
      { "value": "night", "labelEn": "Night", "labelAr": "ليلًا" }
    ] },

  { "id": "q_goals", "questionEn": "What are your fitness goals?",
    "questionAr": "ما هي أهدافك من الاشتراك؟", "type": "MULTI_CHOICE",
    "required": true, "sortOrder": 6,
    "options": [
      { "value": "fat_loss", "labelEn": "Fat Loss", "labelAr": "خسارة الدهون" },
      { "value": "muscle_gain", "labelEn": "Muscle Gain", "labelAr": "زيادة الكتلة العضلية" },
      "..." ] },

  { "id": "q_photo", "questionEn": "Upload a recent photo (optional, for progress tracking)",
    "questionAr": "قم برفع صورة حديثة (اختياري لمتابعة التقدم)", "type": "PHOTO",
    "options": null, "required": false, "sortOrder": 7 }
]
```

**Render by `type`:**

| `type` | Screen control | Answer field to send |
| --- | --- | --- |
| `TEXT` | Text input / textarea | `value` (string) |
| `NUMBER` | Numeric input | `value` (numeric string, e.g. `"4"`) |
| `BOOLEAN` | نعم/لا toggle | `value`, exactly `"true"` or `"false"` |
| `SINGLE_CHOICE` | Radio group / dropdown, from `options` | `value` = the chosen option's `value` (not its label) |
| `MULTI_CHOICE` | Checkbox group, from `options` | `values` = array of chosen `value`s |
| `PHOTO` | Image picker | `fileUrls` = URLs from `POST /mobile/uploads` |

**Conditional questions** (`showIfQuestionId` set): hide the question
entirely until the referenced question has been answered with exactly
`showIfValue` in the *current, in-progress* form state — this is
evaluated client-side, live, as the member fills the wizard (e.g. the
injury-details textarea only appears once "Do you have any injuries?" is
answered "true"). A conditionally-hidden question is not required even if
`required: true`, and you should simply omit it from the submission.

`labelEn`/`labelAr` on an option are display text only — always send the
option's `value`, never its label.

## 3. `POST /mobile/onboarding` — submit once

```json
{
  "answers": [
    { "questionId": "q_occupation", "value": "Software Engineer" },
    { "questionId": "q_days", "value": "4" },
    { "questionId": "q_injuries", "value": "false" },
    { "questionId": "q_time", "value": "evening" },
    { "questionId": "q_goals", "values": ["fat_loss", "fitness"] },
    { "questionId": "q_photo", "fileUrls": ["https://.../photo.jpg"] }
  ]
}
```

One entry per question that applies — skip optional questions the member
left blank, and skip any question currently hidden by a `showIf` condition.

**Responses:**
- `201` — `{ completedAt, answers: [{ questionId, questionEn, questionAr, type, value, values, fileUrls }] }`.
- `400` — a required-and-visible question has no answer; an unknown/
  inactive `questionId`; or a value doesn't match its question's type
  (non-numeric `NUMBER`, not `"true"`/`"false"` for `BOOLEAN`, a choice
  value not present in that question's `options`). `errors`/`message`
  names the offending question.
- `409` — already completed.
- `401` — no/expired member token.

## 4. `GET /mobile/onboarding` — read back

Same shape as the `201` response above; `404` before the member has
completed it.

---

## Gotchas checklist

1. **Render generically.** Don't hardcode field names, screen counts, or
   Arabic labels in the app — everything comes from
   `GET /mobile/onboarding/questions`, and an admin can add, remove, or
   reword questions at any time without an app update.
2. Call `/status` after **every** login — it's the only signal for
   whether to show the wizard (or use the login response's
   `onboardingCompleted` shortcut).
3. `SINGLE_CHOICE`/`MULTI_CHOICE` answers are the option's `value` (a
   stable id like `"morning"`), never its `labelEn`/`labelAr` display text.
4. Evaluate `showIfQuestionId`/`showIfValue` client-side against the
   in-progress form — there's no server round-trip to check visibility.
5. Photos: upload each image through `POST /mobile/uploads` first, then
   pass the returned URLs in that question's `fileUrls`.
6. One submission only — disable the submit button after a `201`, and
   treat a `409` as "already done, go home".
7. There is no `prefill` in this version (removed along with the fixed
   schema) — don't expect the wizard's first screen to arrive pre-filled
   from the member's account.
8. The admin side (Settings → Onboarding Questions — add/edit/reorder/
   retire questions) is documented for the dashboard team in
   `docs/API.md`'s "Onboarding Questions" section; a member's submitted
   answers also feed Sandy AI's per-member context, so Sandy can reference
   what a member told the app at signup.
