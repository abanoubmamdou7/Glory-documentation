# 04 — Videos Library

> Audience: **web dashboard frontend**. Covers the **Videos Library** nav
> item: the video grid/search screen, **Add New Video**, and **Edit Video**.
> Reads need only a valid token; writes need `videos.manage` (ADMIN bypasses).

Video files are hosted on **Vimeo** — the backend never stores the video
bytes itself, only the resulting URL/metadata. Uploading happens **inline**
as part of Add/Edit: submit the form as `multipart/form-data` with the file
attached, and the server uploads it to Vimeo and saves the record in one
call. This returns `503` until the server's `.env` has `VIMEO_CLIENT_ID` /
`VIMEO_CLIENT_SECRET` / `VIMEO_ACCESS_TOKEN` set (a Personal Access Token
from a Vimeo app at developer.vimeo.com with `upload`/`edit`/`delete`/
`video_files` scopes).

| # | Method & path | Screen |
| --- | --- | --- |
| 1 | `POST /video-library` | Add New Video — final "Add Video" click |
| 2 | `GET /video-library` | The grid + search/filter bar |
| 3 | `GET /video-library/categories` | The "Category" filter/form dropdown |
| 4 | `GET /video-library/:id` | Prefills Edit Video |
| 5 | `PATCH /video-library/:id` | Edit Video — "Save Changes" |
| 6 | `DELETE /video-library/:id` | Card's trash icon |

---

## 1. `POST /video-library` — "Add New Video"

One `multipart/form-data` request carries the file **and** every field —
there's no separate upload step. Submit it exactly when the admin clicks
**Add Video**.

| Field | Type | Required | Screen label |
| --- | --- | --- | --- |
| `video` | file | no* | The "Video" drop-zone |
| `audience` | `MALE` \| `FEMALE` | **yes** | "Type Video" |
| `titleEn` | string ≤150 | **yes** | Title (EN) |
| `titleAr` | string ≤150 | no | Title (AR) |
| `descriptionEn` | string | no | Description (EN) |
| `descriptionAr` | string | no | Description (AR) |
| `category` | string ≤100 | no | Category (free text — see below) |
| `videoUrl` | string | no* | Manual fallback — see below |

\* Either attach a `video` file **or** supply `videoUrl` manually (e.g.
pasting an already-hosted Vimeo link) — one of the two is required. In the
normal flow, always attach the file; the manual `videoUrl` path exists for
edge cases, not the primary UI.

`video` accepts any `video/*` mimetype, up to `VIDEO_MAX_UPLOAD_MB` (server
default 500 MB) — validated by content-type, not extension.

**201:** the created video record —

```json
{
  "id": "...", "audience": "MALE",
  "titleEn": "Resistance Band Chest Press", "titleAr": "...",
  "category": "Upper Body - Chest",
  "videoUrl": "https://player.vimeo.com/video/123456789",
  "vimeoUri": "/videos/123456789",
  "thumbnailUrl": "https://i.vimeocdn.com/video/....jpg",
  "duration": "12:10", "fileSize": "10 MB",
  "createdAt": "...", "updatedAt": "..."
}
```

- `videoUrl` is an **embeddable player URL** — use it directly as an
  `<iframe src>` for the preview box and later playback.
- `thumbnailUrl` / `duration` can come back `null` if Vimeo hasn't finished
  processing the file yet (rare, but possible for very large uploads) — this
  does **not** block the save; the fields are optional everywhere.
- Show real upload progress from this request (the backend streams straight
  through to Vimeo) — there's no separate "processing" step visible beyond
  Vimeo's own transcoding.

**Errors:** `400` no file and no `videoUrl` / wrong file type / too large;
`403` missing `videos.manage`; `503` Vimeo not configured on the server yet.

> `category` is free-text, not a fixed enum — the list page shows both
> workout categories ("Upper Body - Chest") and content categories
> ("Onboarding") through the same field. Populate the dropdown from
> `GET /video-library/categories` **plus** allow typing a new one.

## 2. `GET /video-library` — the grid

Query: `page`, `limit`, `search` (matches video name, `titleEn`/`titleAr`),
`category`, `type` (`MALE`|`FEMALE` — the card's audience badge), `dateFrom`/
`dateTo` (the "Date Added" filter), `sortBy` (`titleEn`|`category`|
`createdAt`), `sortOrder`.

Card mapping: audience badge (top-left, Male/Female) = `audience`; orange
category pill = `category`; duration overlay on the thumbnail = `duration`;
"File Size" / "Date Added" row = `fileSize` / `createdAt`.

## 3. `GET /video-library/categories`

`{ "success": true, "data": ["Onboarding", "Upper Body - Chest", "..."] }` —
distinct values already used across all videos, sorted. Feeds both the list's
Category filter and the Add/Edit form's Category field.

## 4. `GET /video-library/:id`

Same shape as a list item — prefills every field of the Edit Video screen,
including the existing `videoUrl`/`thumbnailUrl` for the preview box.

## 5. `PATCH /video-library/:id` — "Save Changes"

Same request shape as create (`multipart/form-data`), every field optional —
send only what changed. **Only attach `video` if the admin is replacing the
file** — doing so re-uploads to Vimeo and overwrites `videoUrl`/`vimeoUri`/
`thumbnailUrl`/`duration`/`fileSize` together; the old Vimeo asset is **not**
automatically deleted in that case (only a full `DELETE` of the video record
cleans up Vimeo — see below).

## 6. `DELETE /video-library/:id`

Deletes the DB record **and** the underlying Vimeo asset. The Vimeo delete is
best-effort — if Vimeo is unreachable or the asset is already gone, the
record is still removed from the library (logged server-side, never blocks
the response).

---

## Gotchas checklist

1. There is no separate upload endpoint — `POST /video-library` and
   `PATCH /video-library/:id` both accept the `video` file directly as
   multipart. Don't call anything else first.
2. `videoUrl` is a **player embed URL**, not a page link — use it in an
   `<iframe>`, not a redirect/`<a href>`.
3. `thumbnailUrl`/`duration` can legitimately be `null` right after upload —
   don't block the save button on them.
4. `category` has no fixed catalog — combine the dropdown from
   `GET /video-library/categories` with free typing.
5. On Edit, only include the `video` field when actually replacing the file
   — sending the request without it leaves the existing video untouched.
