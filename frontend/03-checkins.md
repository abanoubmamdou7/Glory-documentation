# 03 — Check-ins (front-desk QR scanning)

> Audience: **web dashboard frontend**. One endpoint: the front-desk screen
> that scans a member's mobile-app gym-entry QR code.

The mobile app ([mobile/05-bookings-checkin.md](../mobile/05-bookings-checkin.md))
generates a short-lived QR token on the member's phone
(`POST /mobile/checkin/qr`). This screen scans it and records attendance.

## `POST /checkin/scan`

**Auth:** `checkins.manage` (ADMIN bypasses). Grant this key from the Team
Management permissions tab to any front-desk STAFF who needs to scan.

**Body:**

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `token` | string | **yes** | The raw string decoded from the scanned QR |
| `branchId` | id | no | Defaults to the scanning employee's `primaryBranchId` |

**201:**

```json
{ "member": { "id": "...", "memberCode": "10000", "fullName": "احمد حسام", "avatarUrl": null },
  "checkIn": { "id": "...", "type": "GYM", "dateTime": "...", "branchId": "..." },
  "daysRemaining": 10 }
```

Show the member's name/photo and `daysRemaining` on the scanner screen as
confirmation — the member's own phone also picks this up (it polls its QR's
status and shows its own success screen), so this call's job is just to
record the attendance and give the front desk immediate visual confirmation.

**Errors:**

| Code | Meaning | UI |
| --- | --- | --- |
| `404` | Token doesn't exist (bad scan / garbage QR) | "Invalid QR code" |
| `400` | Already used, or expired (>15s since generated) | "Ask the member to refresh their QR and rescan" |
| `403` | Member account isn't `ACTIVE`, **or** the member has no active subscription | Show the reason; direct to the member's profile / sales |

A member with no active subscription is refused entry via this flow — that's
intentional (no automated gym access without a valid membership). A manual
override, if ever needed, isn't part of this endpoint.
