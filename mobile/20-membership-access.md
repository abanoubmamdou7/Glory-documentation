# 20 — Membership access (Release A-lite)

> Audience: **Flutter app**. Branch `claude/release-a-lite` (not deployed).
> The server decides what a membership allows. The app reads it and shows a
> locked feature with the reason; it never hides a feature and never
> re-derives the rule from dates.

## 1. The five states

| `state` | Means | Gym QR | Gym services | Sandy |
| --- | --- | --- | --- | --- |
| `ACTIVE` | a running membership | ✅ | ✅ | member limits |
| `PENDING_ACTIVATION` | bought, starts on the first visit | ✅ (the first scan activates it) | 🔒 | limited |
| `FROZEN` | paused by the gym | 🔒 | 🔒 | limited |
| `EXPIRED` | ended | 🔒 | 🔒 | limited |
| `FREE` | signed up, never bought (or only voided/cancelled/transferred sales) | 🔒 | 🔒 | limited |

**Guest** (no token) is not a state: every member endpoint answers 401 (see
`01-auth.md` → Guest access). Account status (disabled by the gym) is not a
membership state either: it blocks login (403) before any of this.

## 2. `GET /mobile/membership`

Member Bearer. Call it **after login, on app resume, and after any `403
MEMBERSHIP_REQUIRED` or `429 SANDY_QUOTA_EXCEEDED`**.

```json
{
  "state": "ACTIVE",
  "subscription": {
    "id": "cm…", "packageNameEn": "Gym 3 Months", "packageNameAr": "جيم ٣ شهور",
    "membershipType": "GYM",
    "startDate": "2026-11-01T00:00:00.000Z", "endDate": "2027-02-01T00:00:00.000Z",
    "daysRemaining": 73
  },
  "pendingSubscription": null,
  "frozenUntil": null,
  "lastEndedAt": null,
  "lockedReason": null,
  "canUse": {
    "gymQr": true, "ptCheckIn": true, "classEntry": true, "workoutLogging": true,
    "newWorkoutPrograms": true, "newNutritionPlans": true, "coachChatSend": true,
    "sandyChat": true, "sandyDocuments": true
  },
  "sandy": {
    "group": "MEMBER",
    "daily":   { "limit": 50, "used": 3, "remaining": 47, "resetsAt": "2026-11-20T21:00:00.000Z" },
    "monthly": { "limit": null, "used": 31, "remaining": null, "resetsAt": "2026-11-30T21:00:00.000Z" }
  }
}
```

| Field | Rules |
| --- | --- |
| `subscription` | ACTIVE → the running one; FROZEN → the frozen one; EXPIRED → the last ended one; PENDING_ACTIVATION and FREE → `null` |
| `subscription.startDate` | the real start for a membership activated on the first visit |
| `subscription.daysRemaining` | a number only for a running membership; otherwise `null` |
| `pendingSubscription` | the sale waiting for its first visit. Also set for an ACTIVE member whose renewal waits (state stays ACTIVE) |
| `frozenUntil` | FROZEN only: `YYYY-MM-DD` (Amman) when the freeze ends; `null` if open-ended |
| `lastEndedAt` | EXPIRED only |
| `lockedReason` | `null` when ACTIVE; otherwise `{ "en", "ar" }` explaining why member features are locked — the same text a `403 MEMBERSHIP_REQUIRED` would carry for that state. Show it on locked buttons / the membership card. (Added 2026-11-21; additive.) |
| `membershipType` | `GYM`, `PT`, `CLASS`, `APP`, `ASSESSMENT` — every type counts as membership |
| `sandy.*.limit` / `remaining` | `null` = no limit |
| `sandy.*.resetsAt` | ISO instant of Amman midnight (day) / the 1st of next month 00:00 Amman |

### `canUse` → what to lock

| Flag | True when | Unlocks |
| --- | --- | --- |
| `gymQr` | ACTIVE or PENDING_ACTIVATION | the gym entry QR button |
| `ptCheckIn` | ACTIVE | PT check-in button and the session-QR scanner |
| `classEntry` | ACTIVE | class check-in |
| `workoutLogging` | ACTIVE | "اضافة وزن" |
| `newWorkoutPrograms`, `newNutritionPlans` | ACTIVE | information only: show "Renew to get new programmes / plans" when false |
| `coachChatSend` | ACTIVE | the chat message box (history stays readable) |
| `sandyChat` | quota left in both windows | the Sandy message box |
| `sandyDocuments` | ACTIVE | "Upload medical file" in Sandy |

Per item, `canCheckIn` on a booking now also requires ACTIVE.

## 3. `403 MEMBERSHIP_REQUIRED`

Always and only "the membership does not allow this". Nothing was written:
no check-in, no session used, no code consumed, no message stored.

```json
{ "success": false, "statusCode": 403, "error": "MEMBERSHIP_REQUIRED",
  "membershipState": "FROZEN", "action": "GYM_ENTRY",
  "message": "…", "messageEn": "…", "messageAr": "…" }
```

`message` follows the `Accept-Language` header (`ar…` → Arabic, anything else
→ English). `messageEn` and `messageAr` are always both present.

| `membershipState` | English | Arabic |
| --- | --- | --- |
| EXPIRED | Your membership has ended. Renew your membership to use this feature. | انتهى اشتراكك. جدّد اشتراكك لتتمكن من استخدام هذه الميزة. |
| FROZEN | Your membership is frozen. This feature will be available again when your membership resumes on {date}. | اشتراكك مجمّد حالياً. ستعود هذه الميزة متاحة عند استئناف اشتراكك بتاريخ {date}. |
| FROZEN, no end | Your membership is frozen. This feature will be available again when your membership resumes. | اشتراكك مجمّد حالياً. ستعود هذه الميزة متاحة عند استئناف اشتراكك. |
| FREE | This feature is available to members only. Subscribe to a membership to use it. | هذه الميزة متاحة للمشتركين فقط. اشترك في إحدى الباقات لاستخدامها. |
| PENDING_ACTIVATION | Your membership starts on your first visit. Check in at reception to activate it. | يبدأ اشتراكك من أول زيارة. سجّل دخولك عند الاستقبال لتفعيله. |

## 4. Complete authorization matrix (every mobile route)

Built from the server's route table (89 HTTP routes + the chat socket), not
from memory. Order on every gated route: not yours → **404**; membership →
**403 `MEMBERSHIP_REQUIRED`**; then the route's own rules.

| Class | Meaning |
| --- | --- |
| **Public** | no token |
| **Account** | any signed-in account, any membership state (incl. FREE) |
| **Read-only** | any account; reads the member's own history |
| **Quota** | any account; metered by Sandy quota (§6) |
| **Paid** | ACTIVE or PENDING_ACTIVATION |
| **Active** | ACTIVE only |

| Route | Class | `action` when refused |
| --- | --- | --- |
| `GET /mobile/app-config`, `GET /mobile/content/pages`, `/pages/:key`, `/faqs`, `/contact`, `GET /mobile/videos`, `/videos/categories`, `/videos/:id` | Public | — |
| `/mobile/auth/*` (register, verify-otp, resend-otp, complete-registration, login, forgot-password, reset-password, refresh) | Public (sign-in) | — |
| `POST /mobile/auth/logout`, `GET /mobile/auth/me` | Account | — |
| `GET /mobile/membership` | Account | — |
| `GET/PATCH /mobile/profile`, `PATCH /profile/notifications`, `POST /profile/change-password`, `/change-email/request`, `/change-email/confirm`, `POST /profile/delete-account/request`, `DELETE /delete-account/confirm` | Account | — |
| `GET/POST /mobile/family`, `PATCH/DELETE /family/:id` | Account | — |
| `GET /mobile/onboarding`, `/status`, `/questions`, `POST /mobile/onboarding` | Account | — |
| `GET /mobile/notifications`, `/:id`, `/unread-count`, `GET/PATCH /preferences`, `PATCH /read-all` | Account | — |
| `GET/POST/DELETE /mobile/devices` | Account | — |
| `POST /mobile/feedback`, `POST /mobile/uploads` | Account | — |
| `GET /mobile/assessment/questions` | Account | — |
| `GET /mobile/sandy/suggestions`, `GET /mobile/sandy/conversations`, `/:id/messages`, `PATCH/DELETE /conversations/:id` | Account | — |
| `GET/PATCH /mobile/sandy/nudges` (proactive-message mute) | Account | — |
| `GET /mobile/sandy/documents`, `/:id`, `DELETE /documents/:id` (own uploaded files) | Account | — |
| `PATCH /mobile/bookings/:id/cancel`, `POST /bookings/:id/rate` (an attended session) | Account | — |
| `PATCH /mobile/chat/conversations/:id/read` | Account | — |
| `GET /mobile/subscriptions`, `/subscriptions/current`, `GET /mobile/body-records` | Read-only | — |
| `GET /mobile/inbody`, `/summary`, `/trends`, `/:id` | Read-only | — |
| `GET /mobile/bookings`, `/:id`, `/:id/rating` | Read-only | — |
| `GET /mobile/workouts`, `/:id`, `/:id/videos` | Read-only | — |
| `GET /mobile/nutrition-plans`, `/current`, `/:id` | Read-only | — |
| `GET /mobile/chat/conversations`, `/:id/messages`, `/unread-count` | Read-only | — |
| `GET /mobile/group-classes` (timetable) | Read-only | — |
| `GET /mobile/checkin/qr/:id` (poll a code already issued) | Read-only | — |
| `POST /mobile/sandy/chat`, `/chat/stream` | Quota | 429 `SANDY_QUOTA_EXCEEDED` |
| `POST /mobile/checkin/qr` | **Paid** | `GYM_ENTRY` |
| `POST /mobile/bookings/:id/check-in` | **Active** | `PT_CHECK_IN` (PT/appointment), `CLASS_ENTRY` (class), `GYM_ENTRY` (gym booking) |
| `POST /mobile/session-checkin/scan` | **Active** | `PT_CHECK_IN` |
| `POST /mobile/workouts/:id/weight` | **Active** | `WORKOUT_LOG` |
| socket `/chat` → `message:send` (member) | **Active** | `COACH_CHAT` (socket `error` event) |
| `POST /mobile/sandy/documents` | **Active** | `SANDY_DOCUMENT` |
| `POST /mobile/auth/review-signup` | retired | 403 `REVIEW_SIGNUP_RETIRED` |

**Every member action that uses a gym service is gated**: gym entry, PT and
class check-in (button and QR), workout logging, coach chat sending, Sandy
medical files, Sandy chat (quota). The Account routes are the member's own
profile, settings, devices and personal data, which every account must keep
(App Store rules require profile and deletion to work for every account).
Cancelling a booking and rating a session already attended are allowed in
every state on purpose: they close out history, they do not use a service.

**Pending members** can do exactly one thing more than FREE/EXPIRED/FROZEN:
get the gym QR (and be activated by the reception scan, §5). Everything
marked Active is locked for them; Sandy uses the LIMITED quota.

### Deferred (not in A-lite; none are member-callable)

| Item | Where | Today |
| --- | --- | --- |
| Staff-side gates + override permission/audit | desk `POST /checkins` CLASSES/APPOINTMENTS, coach check-in `POST /coach-sessions/:id/check-in`, `POST /coach-sessions` | staff decide; no membership check |
| Coach sending to a lapsed member | socket `message:send` (coach side) | allowed |
| Staff assigning new workouts / nutrition plans, the automatic nutrition draft after InBody | dashboard | allowed for any member (`newWorkoutPrograms`/`newNutritionPlans` are information only) |
| Item flags `canJoin` (classes) and `canSend` (conversations) | timetable, chat list | not sent; use `canUse.classEntry` / `canUse.coachChatSend` |
| Dashboard-editable Sandy limits | `/settings/sandy-limits` | limits come from server environment variables |
| Which package types count as membership | decision D3 | every type counts (see the handoff) |

## 5. Gym entry and the pending membership

- A PENDING_ACTIVATION member gets a gym QR like an active member.
- **The first gym entry activates the membership**, in the same transaction
  as the entry: reception scanning the QR, or reception recording a gym entry
  at the desk. The membership starts on that **Amman calendar day**, and its
  end date is recomputed from the package duration.
- It happens **once**. A second scan the same day is refused (`409`, `error:
  "GYM_ALREADY_CHECKED_IN_TODAY"`, one gym entry per Amman day; the body also
  keeps the older `code` field with the same value for the dashboard) and changes
  nothing; two scans at the same instant produce one entry and one activation.
- The outstanding-balance policy runs first: a refused entry activates nothing.
- A PT session, a class, or a booking check-in never activates a membership
  (they need ACTIVE). A pending member must enter through reception first.
- An ACTIVE member with a renewal waiting: the entry uses the running
  membership; the renewal stays pending.
- If several sales are pending, the oldest is activated.
- After the success screen, call `GET /mobile/membership` (state is now ACTIVE).

`GET /mobile/checkin/qr/:id` is unchanged (`PENDING` / `EXPIRED` /
`CONSUMED`, with `daysRemaining` once consumed).

## 6. Sandy quota: `429 SANDY_QUOTA_EXCEEDED`

| Group | Who | Per Amman day | Per month |
| --- | --- | --- | --- |
| `LIMITED` | FREE, PENDING_ACTIVATION, FROZEN, EXPIRED (and any account with no membership) | 5 | 50 |
| `MEMBER` | ACTIVE only | 50 | no limit (none by default; the server can set one) |

(Server defaults; set by environment variables, see the handoff.)

- Counted: each message the member sends on `POST /mobile/sandy/chat` or
  `/chat/stream` that is accepted. Deleting conversations does not give
  messages back. Medical document uploads do not count.
- Daily window resets at **00:00 Asia/Amman**, monthly on the 1st at 00:00
  Amman, computed from the time-zone rules (correct across daylight-saving
  changes). Use `resetsAt`, never a local calculation.
- A refused message is neither stored nor counted. Parallel sends cannot go
  past the limit.
- On `/chat/stream` the 429 is a **normal HTTP 429 JSON response**, returned
  before any stream opens.

```json
{ "success": false, "statusCode": 429, "error": "SANDY_QUOTA_EXCEEDED",
  "period": "DAY", "limit": 5, "used": 5, "resetsAt": "2026-11-20T21:00:00.000Z",
  "message": "…", "messageEn": "…", "messageAr": "…",
  "upgradeHintEn": "Subscribe to a membership to chat with Sandy more.",
  "upgradeHintAr": "اشترك في إحدى الباقات لتحصل على رسائل أكثر مع ساندي." }
```

| `period` | English | Arabic |
| --- | --- | --- |
| DAY | You've used all {limit} Sandy messages for today. You can chat again after midnight (Amman time). | استخدمت كل رسائل ساندي المتاحة لليوم ({limit} رسائل). يمكنك المتابعة بعد منتصف الليل بتوقيت عمّان. |
| MONTH | You've used all {limit} Sandy messages for this month. Your messages renew at the start of next month. | استخدمت كل رسائل ساندي المتاحة لهذا الشهر ({limit} رسالة). تتجدد رسائلك مع بداية الشهر القادم. |

`upgradeHintEn` / `upgradeHintAr` are `null` for the MEMBER group.

## 7. Status codes, one meaning each

| Answer | Meaning | App action |
| --- | --- | --- |
| 401 `TOKEN_INVALID` | token missing / expired / not a member token | refresh once, else login |
| 401 `REFRESH_TOKEN_INVALID` | refresh failed | login |
| 401 / 403 `ACCOUNT_DISABLED`, 401 `ACCOUNT_DELETED` | the account itself | sign out with the message |
| 403 `MEMBERSHIP_REQUIRED` | membership | show locked state, refresh `/mobile/membership` |
| 429 `SANDY_QUOTA_EXCEEDED` | Sandy quota | lock the box until `resetsAt` |
| 429 `OTP_COOLDOWN` | a code was requested too soon | wait `retryAfterSeconds` |
| 403 with no `error` | another rule (another member's session QR, outstanding balance at reception) | show `message` |

The full list of auth codes is in `01-auth.md`.
