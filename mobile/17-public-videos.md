# Public video library (no login)

**New 2026-10-19.** The video library is now readable with **no token at all** —
no member account, no membership. Use it for a browse/discover screen you can
show before anyone signs in.

| | |
| --- | --- |
| `GET /mobile/videos` | published videos, paginated, newest first |
| `GET /mobile/videos/categories` | the categories in use, for a filter row |
| `GET /mobile/videos/:id` | one video |

**Send no `Authorization` header.** These three are public. Sending a member
token is harmless but pointless — the response is identical for everyone.

> ⚠ **Don't confuse this with the clips inside a workout.**
> `GET /mobile/workouts/:id/videos` (member token) returns the videos attached
> to *that member's assigned programme*, in stage order — that is the training
> screen. The endpoints here are the whole public library, unrelated to any
> assignment. Both can exist in the same app.

## The list

```
GET /mobile/videos?page=1&limit=20&search=صدر&category=Chest&audience=MALE
```

| Query | Meaning |
| --- | --- |
| `page`, `limit` | standard pagination (`limit` max 100) |
| `search` | matches the **English or Arabic** title |
| `category` | exact match; take values from `/mobile/videos/categories` |
| `audience` | `MALE` or `FEMALE`. **Omit it to get everything** — there is no server-side default, because a public viewer has no profile |

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "cmv…",
        "titleEn": "Bench Press",
        "titleAr": "بنش برس",
        "descriptionEn": "…",
        "descriptionAr": "…",
        "category": "Upper Body - Chest",
        "audience": "MALE",
        "videoUrl": "https://player.vimeo.com/video/123456789",
        "thumbnailUrl": "https://i.vimeocdn.com/…",
        "duration": "02:10",
        "createdAt": "2026-10-01T…"
      }
    ],
    "meta": { "page": 1, "limit": 20, "total": 37, "totalPages": 2 }
  }
}
```

**Every field is nullable except `id`, `titleEn`, `videoUrl` and `createdAt`.**
A video may genuinely have no Arabic title, no description, no category, no
thumbnail and no duration — the gym fills those in when it feels like it. Render
around that instead of assuming they are present:

- no `titleAr` → fall back to `titleEn` (don't show an empty row)
- no `thumbnailUrl` → your own placeholder
- no `duration` → hide the badge rather than printing `null`
- no `category` → the video still appears in the unfiltered list, it just isn't
  in any category chip

## Playing it

`videoUrl` is a **Vimeo player URL**, meant for a web view or a player that
takes a URL — not a raw `.mp4` you can hand to a native video element. It is the
same URL the dashboard uses.

## Sorting

Fixed: newest first. There is no `sortBy` — a public list has no screen that
needs another order, so the parameter doesn't exist. Sort client-side if you want
something else on a page you already fetched.

## Three things to get right

1. **Only published videos are returned.** A video being uploaded, one whose
   upload failed, and one the gym retired are all invisible here. You will
   therefore see **fewer** videos than a staff member sees in the dashboard —
   that is correct, not a bug, and nothing you can query around.
2. **`404` means "not published", not only "wrong id".** `GET /mobile/videos/:id`
   answers `404` both for an unknown id and for a real video that isn't
   published. If a deep link stops working, the gym retired that video — show a
   friendly "this video is no longer available" and send the user back to the
   list rather than treating it as a crash.
3. **Don't cache the list for long.** The gym adds and retires videos from the
   dashboard with no app release; a long cache means a user taps a video that
   404s. Re-fetch on screen focus, or cache for minutes rather than days.

## What is deliberately not here

The operational fields the dashboard has — the internal Vimeo resource path, the
file size, the upload status, the owning branch — are not in the response. Don't
build anything that expects them. There is also no "related videos", no view
counter and no favourites: nothing records who watched what, because these
endpoints have no idea who is calling.
