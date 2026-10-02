# 14 — Nutrition plans from Sandy (2026-10-03)

> Audience: **mobile app (Flutter)**.

After each InBody test, Sandy prepares a nutrition plan based on how the
member's body changed, and **their coach reviews it first**. The member only
ever receives a plan the coach approved (possibly edited).

## How it reaches the member — no work needed for the basics
When the coach approves, the backend automatically:
1. creates a **new Sandy conversation** titled **"🥗 نظامك الغذائي — <date>"**
   with the full plan as Sandy's message (plain text, emoji headers — render
   it like any Sandy message, no Markdown);
2. adds an **inbox notification** ("🥗 نظامك الغذائي الجديد جاهز").

So the existing Sandy chat list + notifications screens already show it.

## Optional: a dedicated "My nutrition plan" screen
| Endpoint | Returns |
| --- | --- |
| `GET /mobile/nutrition-plans/current` | the plan in force, or `null` |
| `GET /mobile/nutrition-plans?page=&limit=` | approved plans, newest first |
| `GET /mobile/nutrition-plans/:id` | one plan (404 unless mine + approved) |

```json
{
  "id": "cm…",
  "testDate": "2026-10-02T21:00:00.000Z",
  "approvedAt": "2026-10-03T08:10:00.000Z",
  "coach": { "fullName": "Coach Sara" },
  "coachNote": "ممتاز التقدم، كمّل هيك 💪",
  "sandyConversationId": "cm…",
  "plan": {
    "summary": "وزنك نزل 1.5 كغ وعضلاتك زادت 0.7 كغ…",
    "goal": "RECOMPOSITION",
    "dailyCalories": 2200, "proteinG": 150, "carbsG": 250, "fatG": 65,
    "waterLiters": 3,
    "meals": [{ "name": "الفطور", "time": "08:00", "items": ["شوفان 60 غ…"], "notes": null }],
    "guidelines": ["…"],
    "avoid": ["…"]
  }
}
```

Suggested layout: a calories ring + three macro bars, a water target, meal
cards, then tips / avoid. A "Ask Sandy about it" button can open
`sandyConversationId` — Sandy knows the plan and answers consistently with it.

`goal` labels: `FAT_LOSS` تنزيل دهون · `MUSCLE_GAIN` بناء عضل ·
`RECOMPOSITION` إعادة تركيب الجسم · `MAINTENANCE` محافظة.

## Gotchas
- `current` is `null` until a coach approves the first plan — show an empty
  state ("بعد فحص InBody رح يجهزلك كوتشك نظامك الغذائي").
- Plans waiting for the coach, or rejected ones, never appear — that's by
  design, don't treat an empty list as an error.
- Text is in the member's `appLanguage` at generation time (Jordanian Arabic by
  default).
