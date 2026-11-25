# 32 — App Settings-screen switches

> Audience: **dashboard (web)**. Added 2026-11-25.
> Permission: read `settings.view` or `settings.manage`; change `settings.manage`.

Lets the gym hide rows of the mobile app's **Settings** screen, separately for
members and non-members. It only hides the row in the app: the endpoints and
the member's data behind it stay exactly as they are.

| Item key | Row in the app |
| --- | --- |
| `inbody` | سجل الإنبودي (InBody history) |
| `nutritionPlans` | نظامي الغذائي (My nutrition plan) |
| `bodyRecords` | قياسات الجسم (Body measurements) |
| `subscriptions` | الاشتراكات (Subscriptions) |
| `privateSessions` | حصصي الخاصة (My private sessions) |

Two switches per row:

| Switch | Who it applies to |
| --- | --- |
| `members` | ACTIVE and PENDING_ACTIVATION |
| `nonMembers` | FREE, EXPIRED and FROZEN |

Every switch is **on** (shown) until someone turns it off.

## 1. Read — `GET /settings/mobile-app/settings-items`

```json
{
  "items": [
    { "key": "inbody", "labelEn": "InBody history", "labelAr": "سجل الإنبودي", "members": true, "nonMembers": true },
    { "key": "nutritionPlans", "labelEn": "My nutrition plan", "labelAr": "نظامي الغذائي", "members": true, "nonMembers": false },
    { "key": "bodyRecords", "labelEn": "Body measurements", "labelAr": "قياسات الجسم", "members": true, "nonMembers": true },
    { "key": "subscriptions", "labelEn": "Subscriptions", "labelAr": "الاشتراكات", "members": true, "nonMembers": true },
    { "key": "privateSessions", "labelEn": "My private sessions", "labelAr": "حصصي الخاصة", "members": true, "nonMembers": false }
  ],
  "updatedAt": "2026-11-25T09:12:03.000Z"
}
```

Render one row per item, in this order, with two toggles. Use the labels from
the response rather than hard-coding them.

## 2. Change — `PATCH /settings/mobile-app/settings-items`

Partial: send only what changed. A single toggle flip is a one-item body.

```json
{ "items": [ { "key": "nutritionPlans", "nonMembers": false } ] }
```

Returns the same shape as the GET. `400` for an unknown `key` or a value that
is not `true`/`false`.

The app reads the result on its next `GET /mobile/membership` (field
`settingsItems`), so a change shows without an app release.

## Gotchas

- Hiding a row is not a permission change. Members can still reach the data
  through other screens and the API; nothing is deleted.
- PENDING_ACTIVATION counts as a member (it has a paid sale waiting for its
  first visit). FROZEN counts as a non-member.
- The switches are gym-wide. There is no per-member override.
