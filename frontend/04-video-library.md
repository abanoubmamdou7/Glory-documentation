# 04 — Videos Library

> Audience: **web dashboard frontend**. Covers the **Videos Library** nav
> item: the video grid/search screen, **Add New Video**, and **Edit Video**.
> Reads need only a valid token; writes need `videos.manage` (ADMIN bypasses).

Video files are hosted on **Vimeo** — the backend never stores the video
bytes itself, only the resulting URL/metadata. `POST /video-library/upload`
returns `503` until the server's `.env` has `VIMEO_CLIENT_ID` /
`VIMEO_CLIENT_SECRET` / `VIMEO_ACCESS_TOKEN` set (a Personal Access Token
from a Vimeo app at developer.vimeo.com with `upload`/`edit`/`delete`/
`video_files` scopes).

| # | Method & path | Screen |
| --- | --- | --- |
| 1 | `POST /video-library/upload` | The "Video" drop-zone inside Add/Edit |
| 2 | `POST /video-library` | Add New Video — final "Add Video" click |
| 3 | `GET /video-library` | The grid + search/filter bar |
| 4 | `GET /video-library/categories` | The "Category" filter/form dropdown |
| 5 | `GET /video-library/:id` | Prefills Edit Video |
| 6 | `PATCH /video-library/:id` | Edit Video — "Save Changes" |
| 7 | `DELETE /video-library/:id` | Card's trash icon |

---

## The upload step

The form's "Video" box uploads **before** the rest of the form is submitted —
call this as soon as a file is dropped/selected, show its own progress state,
then let the admin fill in the text fields and hit **Add Video**/**Save
Changes** once it's done (exactly like `avatarUrl` coming from
`POST /uploads` in the Team wizard, just a separate endpoint since this one
talks to Vimeo instead of Cloudinary and accepts much larger files).

### `POST /video-library/upload`

Multipart, field **`video`**. Any `video/*` mimetype, up to
`VIDEO_MAX_UPLOAD_MB` (server default 500 MB) — validated by content-type,
not extension.

**201:**

```json
{
  "videoUrl": "https://player.vimeo.com/video/123456789",
  "vimeoUri": "/videos/123456789",
  "thumbnailUrl": "https://i.vimeocdn.com/video/....jpg",
  "duration": "12:10",
  "fileSize": "10 MB",
  "originalName": "chest-press.mp4"
}
```

- `videoUrl` is an **embeddable player URL** — use it directly as an
  `<iframe src>` for the preview box and later playback.
- `thumbnailUrl` / `duration` can come back `null` if Vimeo hasn't finished
  processing the file yet (rare, but possible for very large uploads) — this
  does **not** block saving the video; the fields are optional everywhere.
- Keep all five returned fields (plus `videoUrl`) and send them straight
  through on the create/update call below — don't try to recompute or edit
  them client-side.

**Errors:** `400` no file / wrong type / too large; `403` missing
`videos.manage`; `503` Vimeo not configured on the server yet.

---

## 2. `POST /video-library` — "Add New Video"

| Field | Type | Required | Screen label |
| --- | --- | --- | --- |
| `audience` | `MALE` \| `FEMALE` | **yes** | "Type Video" |
| `titleEn` | string ≤150 | **yes** | Title (EN) |
| `titleAr` | string ≤150 | no | Title (AR) |
| `descriptionEn` | string | no | Description (EN) |
| `descriptionAr` | string | no | Description (AR) |
| `category` | string ≤100 | no | Category (free text — see below) |
| `videoUrl` | string | **yes** | From the upload step |
| `vimeoUri` | string | no | From the upload step |
| `thumbnailUrl` | string | no | From the upload step |
| `duration` | string | no | From the upload step, e.g. `"12:10"` |
| `fileSize` | string | no | From the upload step, e.g. `"10 MB"` |

**201:** the created video record (same shape as `GET /video-library/:id`).

> `category` is a free-text field, not a fixed enum — the list page shows
> both workout categories ("Upper Body - Chest") and content categories
> ("Onboarding") through the same field. Populate the dropdown from
> `GET /video-library/categories` **plus** allow typing a new one (the design
> shows both existing and freshly-typed categories on cards).

## 3. `GET /video-library` — the grid

Query: `page`, `limit`, `search` (matches video name, `titleEn`/`titleAr`),
`category`, `type` (`MALE`|`FEMALE` — the card's audience badge), `dateFrom`/
`dateTo` (the "Date Added" filter), `sortBy` (`titleEn`|`category`|
`createdAt`), `sortOrder`.

Card mapping: audience badge (top-left, Male/Female) = `audience`; orange
category pill = `category`; duration overlay on the thumbnail = `duration`;
"File Size" / "Date Added" row = `fileSize` / `createdAt`.

## 4. `GET /video-library/categories`

`{ "success": true, "data": ["Onboarding", "Upper Body - Chest", "..."] }` —
distinct values already used across all videos, sorted. Feeds both the list's
Category filter and the Add/Edit form's Category field.

## 5. `GET /video-library/:id`

Same shape as a list item — prefills every field of the Edit Video screen,
including the existing `videoUrl`/`thumbnailUrl` for the preview box (the
admin only needs to re-upload if actually replacing the file).

## 6. `PATCH /video-library/:id` — "Save Changes"

Same fields as create, all optional — send only what changed. If the admin
replaced the file, run the upload step again first and send the new
`videoUrl`/`vimeoUri`/`thumbnailUrl`/`duration`/`fileSize` together; the old
Vimeo asset is **not** automatically deleted in that case (only a full
`DELETE` of the video record cleans up Vimeo — see below).

## 7. `DELETE /video-library/:id`

Deletes the DB record **and** the underlying Vimeo asset. The Vimeo delete is
best-effort — if Vimeo is unreachable or the asset is already gone, the
record is still removed from the library (logged server-side, never blocks
the response).

---

## Gotchas checklist

1. Always call the upload endpoint first — `POST /video-library` /
   `PATCH /video-library/:id` never accept a raw file, only the URL/metadata
   the upload step returned.
2. `videoUrl` is a **player embed URL**, not a page link — use it in an
   `<iframe>`, not a redirect/`<a href>`.
3. `thumbnailUrl`/`duration` can legitimately be `null` right after upload —
   don't block the save button on them.
4. `category` has no fixed catalog — combine the dropdown from
   `GET /video-library/categories` with free typing.
5. Large uploads: show real upload progress from the multipart request (the
   backend streams straight through to Vimeo, so there's no separate
   "processing" step visible to the admin beyond Vimeo's own transcoding).
