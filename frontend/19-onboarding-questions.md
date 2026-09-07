# 15 — Onboarding Questions (Settings)

> Audience: **web dashboard frontend**. No screens were provided for this
> one — built directly from the request ("make the mobile app's first-login
> questions dashboard-configurable, not fixed") and the schema, same as
> Workouts originally was. Place it wherever Settings ends up putting it;
> functionally it's a content-catalog CRUD identical in shape to FAQ /
> Assessment Evaluation Management (§3/§4 of
> [08-company-data.md](08-company-data.md)) — reuse that same list+drawer UI
> pattern if you want visual consistency.

Manages the question catalog the mobile app's first-login intake wizard
renders (`GET /mobile/onboarding/questions` — see
[../mobile/07-onboarding.md](../mobile/07-onboarding.md) for the member
side). **Permission:** `settings.view` (reads) / `settings.manage`
(writes) — the same pair as the rest of Settings, ADMIN bypasses.

## Why this exists

The mobile onboarding wizard used to be a fixed, hardcoded form (44
typed fields across 8 sections). It's now fully dynamic: whatever question
rows exist here (active, in `sortOrder`) are exactly what the app shows —
add, edit, reorder, or retire a question here and the app reflects it on
the member's next login, no release needed.

## The question editor

| Field | Notes |
| --- | --- |
| Question (EN / AR) | Plain text, both required |
| Type | `TEXT` / `NUMBER` / `BOOLEAN` / `SINGLE_CHOICE` / `MULTI_CHOICE` / `PHOTO` — pick one; changes what the rest of the form asks for |
| Options | Only for `SINGLE_CHOICE`/`MULTI_CHOICE` — a repeatable `{ value, labelEn, labelAr }` row. `value` is a short stable code (e.g. `"morning"`) the app stores as the answer; `labelEn`/`labelAr` are what the member sees and can be reworded later without breaking anyone's saved answer |
| Required | Whether the member must answer before submitting |
| Show only if... | Optional: pick another question from the list + the exact answer value that reveals this one (e.g. "Describe your injury" only shows if "Do you have any injuries?" = `true`). Both fields together or neither — `400` if only one is set |
| Active | Inactive questions stop appearing to members immediately, but stay attached to anyone who already answered them |

`sortOrder` drives both the member-facing order and this list's default
sort — support drag-to-reorder via `PUT /onboarding-questions/reorder`
(send every id in the new order; same pattern as Contact Methods).

## Endpoints

- `POST /onboarding-questions` — create.
- `GET /onboarding-questions` — paginated list; `search` (question text),
  `isActive` filter.
- `GET /onboarding-questions/:id`
- `PATCH /onboarding-questions/:id` — true partial.
- `PUT /onboarding-questions/reorder` — `{ orderedIds: [...] }`.
- `DELETE /onboarding-questions/:id` — **409** if a member already
  answered it, or another question's "show only if" still points at it.
  Toggle Active off instead of deleting a question with real answers.

## Sandy AI integration

A member's submitted answers are automatically included in the context
Sandy AI is given for that member on every chat turn — no extra wiring
needed here. A member can ask Sandy things grounded in what they told the
app at signup (goals, injuries, training preferences, etc.).

---

## Gotchas checklist

1. Changing `type` on a question that already has real answers doesn't
   retroactively reshape those answers — they stay whatever shape they
   were saved in. Prefer adding a new question and retiring the old one
   over changing an answered question's type.
2. `options[].value` must stay stable if you want old answers to keep
   matching a current option — rename `labelEn`/`labelAr` freely, but
   changing `value` orphans any answer that used the old one.
3. A "show only if" question can't be deleted while another question
   still depends on it — clear or repoint that dependency first.
4. There's no member-facing edit — once a member submits, it's final on
   their side; only the dashboard's question catalog is editable.
