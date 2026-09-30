# Two coaches per member — gym coach + PT coach

**Delivered 2026-09-29.** A member used to have exactly one coach. Now they
have **two slots**: the general/gym coach they get on joining, and a separate
**personal-training coach** for private sessions.

| Slot | Field | Set by |
| --- | --- | --- |
| `GENERAL` | `Member.assignedInstructorId` | the desk — `POST /chat/assign-instructor` |
| `PT` | `Member.ptInstructorId` | **selling a PT package with an instructor** (automatic), or `POST /chat/assign-instructor` with `type: "PT"` |

Both may point at the same person; usually they don't.

---

## 1. Reading a member's coaches

`GET /members/:id` now returns:

```json
"coaches": {
  "general": { "id": "...", "fullName": "Coach Sara" },
  "pt":      { "id": "...", "fullName": "Coach Ahmed" }
}
```

Either can be `null`. The raw `assignedInstructorId` / `ptInstructorId`
scalars are still on the record, unchanged.

`GET /members/:id/operational-profile` gained a top-level **`ptInstructor`**
`{ id, name, avatarUrl, chatConversationId }` next to the existing
`assignedInstructor` (which still means the **gym** coach — its meaning did
not change, so nothing you already render breaks).

`GET /members/:id/instructor-assignments` now returns **`currentPt`** beside
`current`, and every history row carries **`type`** (`GENERAL` | `PT`) so the
timeline can say *which* coach changed.

## 2. Assigning

```
POST /chat/assign-instructor        (chat.manage)
{ "memberId": "...", "instructorId": "...", "type": "PT", "reason": "bought a PT package" }
```

- **`type` is optional and defaults to `GENERAL`** — every call you make today
  keeps working untouched.
- The response now includes `coachType`, telling you which slot you filled.
- Assigning a slot opens (or reuses) the chat with that coach. Reassigning a
  slot leaves the other slot alone and **keeps** the old conversation.

**You usually don't need this for PT.** Selling a package whose
`membershipType` is `PT` with an `instructorId` assigns the PT coach and opens
the chat automatically, in the sale's own transaction — so a PT client can
never exist without a coach.

Rules that protect the desk's decisions:

- A **gym/class sale** that names a trainer does **not** make them the
  member's coach. Only PT sales assign.
- A **renewal or upgrade** copies the old sale's trainer forward, which is not
  a fresh decision — it fills the PT slot only if it is empty. It will **not**
  undo a manual reassignment made in the meantime.
- A **clerical correction** — `PATCH /subscriptions/:id { instructorId }` on a
  PT sale — *does* move the member's PT coach, so the sale and the profile
  never disagree.
- The PT slot is **not cleared when the PT subscription expires**. Show it
  with the subscription's own status if you need to distinguish "current
  client" from "former client".

## 3. Chat: expect two threads

`GET /mobile/chat/conversations` (member) and `GET /coach-chat/conversations`
(coach) now return **`coachTypes`** on every row:

| `coachTypes` | Meaning | Suggested label |
| --- | --- | --- |
| `["GENERAL"]` | the gym coach | "كوتش الجيم" |
| `["PT"]` | the personal trainer | "كوتش التمارين الخاصة" |
| `["GENERAL","PT"]` | one person holds both slots | either, no duplicate row |
| `[]` | a **former** coach, history kept | "كوتش سابق" (read-only in the UI if you prefer) |

⚠ **The mobile app currently assumes a single thread** (`conversations[0]`).
That has to become a list — a member with a separate PT coach legitimately has
two. Nothing else about the chat changed: sending is still Socket.IO, the
ownership rules are the same, and the unique key was always
(member, instructor).

On the coach side, `coachTypes` lets the portal split "my gym members" from
"my PT clients" without another request. `GET /instructor-reports/members`
also counts PT clients now — a member reaches a coach's list through the gym
slot, **the PT slot**, an active subscription's trainer, or a workout
assignment.

## 4. Existing members

A one-off backfill ran on deploy: every member with a live (`ACTIVE`/`FROZEN`)
PT subscription got their PT slot set from that sale's trainer, and the chat
thread created if it didn't exist. Members whose slot was already set by hand
were left alone. So the feature is populated from day one — you don't have to
re-assign anyone.

## Gotchas checklist

- [ ] Render `coaches.general` and `coaches.pt` separately — don't collapse them.
- [ ] Chat screens iterate the conversation list; label from `coachTypes`.
- [ ] `assignedInstructor` still means the **gym** coach everywhere.
- [ ] Don't send `type` unless you mean PT — omitting it is GENERAL.
- [ ] A PT sale form's "instructor" field is now a real assignment, not just
      attribution. Say so in the UI.
