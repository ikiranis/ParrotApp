# ParrotApp API Documentation

REST API for ParrotApp — a personal media library manager.

**Base URL:** `http://localhost:9999`
**Content-Type:** `application/json` (unless noted otherwise)

---

## Table of Contents

1. [Error Responses](#error-responses)
2. [Authentication & Authorization](#authentication--authorization)
3. [Users](#users)
4. [Common Schemas](#common-schemas)
5. [General](#general)
6. [Languages](#languages)
7. [Settings](#settings)
8. [Library Folders](#library-folders)
9. [Background Scan](#background-scan)
10. [Photos](#photos)
11. [Videos](#videos)
12. [Music](#music)
13. [Playlists](#playlists)
14. [Smart Playlists](#smart-playlists)
15. [Folders](#folders)
16. [Search](#search)
17. [Thumbnails](#thumbnails)
18. [Media Hashes](#media-hashes)
19. [Uploads](#uploads)
20. [Tag Export and Import](#tag-export-and-import)
21. [Filesystem](#filesystem)
22. [Logs](#logs)
23. [Radio](#radio)

---

## Error Responses

All errors are returned as a JSON object with a consistent shape.

**Schema**

| Field           | Type     | Description                            |
|-----------------|----------|----------------------------------------|
| `status`        | integer  | Numeric HTTP status code               |
| `message`       | string   | Human-readable error description       |
| `httpStatus`    | string   | HTTP status name (e.g. `NOT_FOUND`)    |
| `zonedDateTime` | string   | ISO-8601 timestamp of the error        |

**HTTP status codes used**

| Status | Trigger                                       |
|--------|-----------------------------------------------|
| 400    | Validation failure, invalid input             |
| 401    | Missing or invalid authentication token       |
| 403    | Authenticated user lacks the required role    |
| 404    | Resource not found                            |
| 409    | Conflict (e.g. scan already running, username taken) |
| 413    | Upload larger than the configured multipart limits (`spring.servlet.multipart.max-file-size` / `max-request-size`); rejected by the servlet container before any controller sees it |

**Example**

```json
{
  "status": 404,
  "message": "Photo not found: 42",
  "httpStatus": "NOT_FOUND",
  "zonedDateTime": "2024-03-15T10:30:00+02:00"
}
```

---

## Authentication & Authorization

The API is secured with **stateless JWT bearer tokens**. A client obtains a token by registering the
first user or logging in, then sends it on every subsequent request:

```text
Authorization: Bearer <token>
```

Tokens are signed by the server (HMAC-SHA) and carry the user's username and role. They expire after
24 hours by default (configurable via `app.jwt.expiration-ms`).

### Roles and required permissions

Every user holds one role: `ADMIN` or `USER`. Endpoints are authorized as follows.

| Endpoint group                                              | Required permission        |
|-------------------------------------------------------------|----------------------------|
| `GET /api/auth/status`, `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/general/appAlive` | Public (no token)          |
| All read endpoints (`GET /api/**`)                          | Any authenticated user     |
| `POST /api/photos/{id}/view`, `PATCH /api/photos/{id}/rating` | Any authenticated user (per-user viewing interactions) |
| `POST /api/music/{id}/view`, `PATCH /api/music/{id}/rating` | Any authenticated user (per-user playback interactions) |
| Playlists (`/api/playlists/**`) — every method              | Any authenticated user (own playlists only) |
| Smart playlists (`/api/smart-playlists/**`) — every method, including the search `POST` | Any authenticated user (own smart playlists only) |
| User management (`/api/users/**`)                           | `ADMIN`                    |
| Application log (`/api/logs/**`) — every method             | `ADMIN`                    |
| Tag export (`/api/exports/**`) — every method               | `ADMIN` (an export carries every user's ratings and playlists) |
| All other mutating requests (scanning, library folders, settings, deletes, uploads, tag import, thumbnails, rehash) | `ADMIN` |

Requests without a valid token to a protected endpoint return **401**; authenticated requests
lacking the required role return **403**.

### First-run registration

When the application has no users, the frontend shows a registration screen. The first account
created is always assigned the `ADMIN` role. Once any user exists, registration is closed
(`POST /api/auth/register` returns **409**) and further accounts are created by an administrator via
the Users endpoints.

### GET /api/auth/status

Reports whether the application still needs its first user. Public.

**Response** `200 OK`

```json
{ "needsRegistration": true }
```

### POST /api/auth/register

Creates the first user as an administrator. Only succeeds while no users exist. Public.

**Request body**

| Field      | Type   | Description                          |
|------------|--------|--------------------------------------|
| `username` | string | Desired login name (3–50 chars)      |
| `password` | string | Password (6–100 chars)               |

**Response** `200 OK` — an `AuthResponse` (see below).
**Errors:** `400` invalid body, `409` a user already exists.

### POST /api/auth/login

Authenticates a user and issues a token. Public.

**Request body**

| Field      | Type   | Description    |
|------------|--------|----------------|
| `username` | string | Login name     |
| `password` | string | Password       |

**Response** `200 OK` — an `AuthResponse`.
**Errors:** `401` invalid credentials.

### GET /api/auth/me

Returns the currently authenticated user. Requires a valid token.

**Response** `200 OK` — a `UserDTO` (see [Users](#users)).

### AuthResponse schema

| Field      | Type   | Description                                   |
|------------|--------|-----------------------------------------------|
| `token`    | string | Signed JWT to send via the Authorization header |
| `username` | string | Authenticated user's login name               |
| `role`     | string | `ADMIN` or `USER`                             |

---

## Users

Administrator-only management of user accounts. Every endpoint in this section requires the
`ADMIN` role; a non-admin receives **403**.

### UserDTO schema

| Field      | Type   | Description       |
|------------|--------|-------------------|
| `id`       | long   | Primary key       |
| `username` | string | Login name        |
| `role`     | string | `ADMIN` or `USER` |

### GET /api/users

Lists all user accounts (without password hashes).

**Response** `200 OK` — an array of `UserDTO`.

### POST /api/users

Creates a new user account.

**Request body**

| Field      | Type   | Description                     |
|------------|--------|---------------------------------|
| `username` | string | Desired login name (3–50 chars) |
| `password` | string | Password (6–100 chars)          |
| `role`     | string | `ADMIN` or `USER`               |

**Response** `200 OK` — the created `UserDTO`.
**Errors:** `400` invalid body, `409` username already exists.

### DELETE /api/users/{id}

Deletes the user with the given id.

**Response** `200 OK` on success.
**Errors:** `404` no user with that id.

---

## Common Schemas

### MediaFile

Represents a media file entry in the library.

| Field         | Type    | Description                                               |
|---------------|---------|-----------------------------------------------------------|
| `id`          | long    | Primary key                                               |
| `libraryFolder` | object | The root library folder this file belongs to (see below) |
| `path`        | string  | Directory path relative to the library folder root        |
| `filename`    | string  | File name including extension                             |
| `hash`        | string  | Content hash for de-duplication; `null` if not computed   |
| `kind`        | string  | Media kind: `IMAGE`, `VIDEO`, `AUDIO`, `DOCUMENT`        |
| `thumbnailId` | long    | Primary key of the linked thumbnail; `null` if none       |

### LibraryFolder

| Field  | Type   | Description                                   |
|--------|--------|-----------------------------------------------|
| `id`   | long   | Primary key                                   |
| `path` | string | Full absolute path of the folder on the server |

### DirectoryListing

A listing of one directory on the server filesystem, returned by the folder browser.

| Field         | Type   | Description                                                          |
|---------------|--------|--------------------------------------------------------------------|
| `path`        | string | Absolute path of the listed directory                              |
| `parent`      | string | Absolute path of the parent directory; `null` at a filesystem root |
| `directories` | array  | Immediate subdirectories as `DirectoryEntry` objects (see below)   |

### DirectoryEntry

A single subdirectory within a `DirectoryListing`.

| Field  | Type   | Description                                  |
|--------|--------|----------------------------------------------|
| `name` | string | Directory display name (its last path segment) |
| `path` | string | Absolute path of the directory on the server |

### Folder

Represents a scanned sub-folder within a library.

| Field           | Type    | Description                                           |
|-----------------|---------|-------------------------------------------------------|
| `id`            | long    | Primary key                                           |
| `libraryFolder` | object  | Parent library folder (see LibraryFolder)             |
| `path`          | string  | Path relative to the library folder root              |
| `hash`          | string  | Content hash derived from file count + total size     |
| `level`         | integer | Nesting depth (0 = root, 1 = direct child, …)        |
| `finished`      | boolean | Whether all files in this folder have been indexed    |
| `lastUpdate`    | string  | ISO-8601 timestamp of the last detected change        |
| `thumbnailId`   | long    | Primary key of the linked thumbnail; `null` if none   |

### PhotoDetailDTO

Full detail view combining a `MediaFile` with its optional tag data.

| Field         | Type    | Description                                   |
|---------------|---------|-----------------------------------------------|
| `id`          | long    | Primary key of the media file                 |
| `path`        | string  | Absolute directory path on the server         |
| `filename`    | string  | File name including extension                 |
| `hash`        | string  | Content hash; `null` if not computed          |
| `kind`        | string  | Media kind string (e.g. `IMAGE`)              |
| `name`        | string  | User-assigned display name; `null` if not set |
| `description` | string  | Free-text description; `null` if not set      |
| `album`       | string  | Album name; `null` if not set                 |
| `filesize`    | long    | File size in bytes; `null` if not set         |
| `width`       | integer | Image width in pixels; `null` if not set      |
| `height`      | integer | Image height in pixels; `null` if not set     |
| `viewCount`   | long    | Number of times the current user viewed this photo |
| `rating`      | integer | Current user's rating 1–5; `null` if not rated |
| `dateTaken`   | string  | ISO-8601 date/time photo was taken            |
| `latitude`    | double  | GPS latitude; `null` if not set               |
| `longitude`   | double  | GPS longitude; `null` if not set              |
| `locationName` | string  | Reverse-geocoded place name (e.g. "Athens, Greece"); `null` if not geotagged or unresolved |
| `cameraMake`  | string  | Camera manufacturer from EXIF; `null` if none |
| `cameraModel` | string  | Camera model from EXIF; `null` if none        |
| `mimeType`    | string  | MIME type (e.g. `image/jpeg`); `null` if none |
| `dateCreated`  | string  | ISO-8601 timestamp of tag record creation     |
| `dateUpdated`  | string  | ISO-8601 timestamp of last tag update         |
| `lastViewDate` | string  | ISO-8601 timestamp of the user's last view; `null` if never viewed |

### VideoDetailDTO

Full detail view for a video, combining a `MediaFile` with its optional `VideoTag` and the current
user's `UserTag`.

| Field             | Type    | Description                                          |
|-------------------|---------|------------------------------------------------------|
| `id`              | long    | Primary key of the media file                        |
| `path`            | string  | Directory path relative to the library folder        |
| `filename`        | string  | File name including extension                         |
| `hash`            | string  | Content hash (middle-slice); `null` if not computed   |
| `kind`            | string  | Media kind string (`VIDEO`)                          |
| `name`            | string  | User-assigned display name; `null` if not set         |
| `description`     | string  | Free-text description; `null` if not set              |
| `album`           | string  | Album name; `null` if not set                         |
| `filesize`        | long    | File size in bytes; `null` if not set                 |
| `width`           | integer | Frame width in pixels; `null` if unknown              |
| `height`          | integer | Frame height in pixels; `null` if unknown             |
| `durationSeconds` | long    | Playback duration in whole seconds; `null` if unknown |
| `codec`           | string  | Video codec name; `null` if unknown                   |
| `frameRate`       | double  | Frames per second; `null` if unknown                  |
| `bitrate`         | long    | Overall bitrate in bits/second; `null` if unknown     |
| `dateTaken`       | string  | ISO-8601 date/time the video was recorded; `null` if unknown |
| `mimeType`        | string  | MIME type (e.g. `video/mp4`); `null` if none          |
| `viewCount`       | long    | Number of times the current user viewed this video    |
| `rating`          | integer | Current user's rating 1–5; `null` if not rated        |
| `dateCreated`     | string  | ISO-8601 timestamp of tag record creation             |
| `dateUpdated`     | string  | ISO-8601 timestamp of last tag update                 |
| `lastViewDate`    | string  | ISO-8601 timestamp of the user's last view; `null` if never viewed |

### MusicDetailDTO

Full detail view for a music track or music video, combining a `MediaFile` with its `MusicTag`, the
linked `Album` (and its `Cover`), and the current user's `UserTag`. The same shape backs the music
list rows, the single-track detail, and each entry of the play queue.

| Field           | Type    | Description                                                     |
|-----------------|---------|-----------------------------------------------------------------|
| `id`            | long    | Primary key of the media file                                   |
| `path`          | string  | Directory path relative to the library folder                   |
| `filename`      | string  | File name including extension                                    |
| `kind`          | string  | Music kind: `MUSIC` (audio track) or `MUSIC_VIDEO`              |
| `songName`      | string  | Song title; falls back to the filename when untagged            |
| `artist`        | string  | Performing artist; `null` if unknown                            |
| `genre`         | string  | Genre; `null` if unknown                                        |
| `album`         | string  | Album name: from the linked `Album`, falling back to the tag's own `albumName` (which is how a music video, having no album row, still reports one) |
| `albumId`       | long    | Primary key of the album; `null` when none                      |
| `year`          | integer | Release year: from the linked `Album`, falling back to the tag's own year; `null` if unknown |
| `coverId`       | long    | Primary key of the album cover image; `null` when none          |
| `thumbnailId`   | long    | Primary key of the media file's thumbnail (music videos); `null` otherwise |
| `trackTime`     | long    | Playback duration in whole seconds; `null` if unknown           |
| `live`          | boolean | Whether this is a live recording                                |
| `source`        | string  | Source medium: `REGULAR`, `LOSSLESS`, `VINYL`, `CDRIP`; a null column reads back as `REGULAR` |
| `trackPosition` | integer | Position within the album; `null` if unknown                    |
| `videoWidth`    | integer | Music-video frame width in pixels; `null` for audio tracks      |
| `videoHeight`   | integer | Music-video frame height in pixels; `null` for audio tracks     |
| `filesize`      | long    | File size in bytes; `null` if unknown                           |
| `mimeType`      | string  | MIME type (e.g. `audio/mpeg`, `video/mp4`); `null` if none      |
| `hash`          | string  | Content hash (full for audio, middle-slice for music video)     |
| `rating`        | integer | Current user's rating 1–5; `null` if not rated                  |
| `viewCount`     | long    | Number of times the current user has played this track          |
| `lastViewDate`  | string  | ISO-8601 timestamp of the user's last play; `null` if never played |
| `dateCreated`   | string  | ISO-8601 timestamp of tag record creation                       |

### Setting

| Field          | Type   | Description                     |
|----------------|--------|---------------------------------|
| `id`           | long   | Primary key                     |
| `settingName`  | string | Unique name identifying the setting |
| `settingValue` | string | Current value                   |

### ScanResult

Returned by synchronous folder scan endpoints.

| Field            | Type    | Description                                           |
|------------------|---------|-------------------------------------------------------|
| `added`          | integer | New media files added to the database                 |
| `skipped`        | integer | Files already indexed and skipped                     |
| `errors`         | integer | Files or folders that could not be processed          |
| `foldersScanned` | integer | Leaf directories that had changes and were scanned    |
| `foldersSkipped` | integer | Leaf directories unchanged and skipped                |
| `removed`        | integer | Orphaned media file records removed by post-scan cleanup |
| `message`        | string  | Human-readable summary                                |

### ScanJobResponse

Returned by background scan endpoints.

| Field                    | Type    | Description                                           |
|--------------------------|---------|-------------------------------------------------------|
| `jobId`                  | string  | UUID of the job; `null` when no scan has run          |
| `status`                 | string  | `IDLE`, `RUNNING`, `COMPLETED`, `CANCELLED`, `FAILED` |
| `phase`                  | string  | `COLLECTING`, `SCANNING`, `TAGGING`, `CLEANING`; `null` when idle |
| `startedAt`              | string  | ISO-8601 start time; `null` if idle                   |
| `completedAt`            | string  | ISO-8601 completion time; `null` if running           |
| `added`                  | integer | Media files added so far                              |
| `skipped`                | integer | Files already indexed and skipped                     |
| `errors`                 | integer | Files or folders that produced an error               |
| `foldersScanned`         | integer | Leaf directories fully scanned                        |
| `foldersSkipped`         | integer | Leaf directories skipped as unchanged                 |
| `tagged`                 | integer | Files whose tags have been read (Phase 3)             |
| `removed`                | integer | Orphaned files removed by cleanup (Phase 4)           |
| `totalFolders`           | integer | Total leaf directories discovered in Phase 1          |
| `totalFiles`             | integer | Total new files to tag discovered in Phase 2          |
| `totalMediaFilesInLibrary` | integer | Total media files on filesystem after Phase 1       |
| `totalFoldersToClean`    | integer | Changed folders to inspect during orphan cleanup      |
| `foldersClean`           | integer | Changed folders whose cleanup has completed           |
| `progressPercent`        | integer | Estimated progress in the range [0, 100]              |
| `errorLogs`              | array   | Ordered list of error message strings                 |
| `initialFilesCount`      | integer | Media files in the database before this scan started  |
| `message`                | string  | Human-readable status message                         |

### TagExportItemDTO

| Field      | Type    | Description                                                    |
|------------|---------|----------------------------------------------------------------|
| `hash`     | string  | Content hash of the media file the tag relates to              |
| `username` | string  | Login name of the user the tag belongs to                      |
| `rating`   | integer | User rating 1–5; `null` if not rated                           |
| `views`    | long    | View count; `null` treated as 0 on import                      |

### DatabaseStatsResponse

| Field            | Type | Description                                   |
|------------------|------|-----------------------------------------------|
| `totalFiles`     | long | Total media file records across all kinds      |
| `images`         | long | Media files classified as IMAGE                |
| `videos`         | long | Media files classified as VIDEO                |
| `audioFiles`     | long | Media files classified as AUDIO                |
| `documents`      | long | Media files classified as DOCUMENT             |
| `tags`           | long | Total photo tag records                        |
| `userTags`       | long | Total per-user tag records                     |
| `rated`          | long | User tags that carry a user-assigned rating    |
| `totalViews`     | long | Sum of all view counts across every user tag   |
| `thumbnails`     | long | Total thumbnail records                        |
| `folders`        | long | Total scanned folder records                   |
| `libraryFolders` | long | Total configured library folder roots          |
| `users`          | long | Total user accounts                            |

### HashStatsResponse

| Field    | Type | Description                              |
|----------|------|------------------------------------------|
| `hashed` | long | Image files that have a content hash     |
| `total`  | long | Total image files in the library         |

### MediaHashStatsResponse

Hash coverage for every media category at once; each value is a `HashStatsResponse`.

| Field         | Type   | Description                                        |
|---------------|--------|----------------------------------------------------|
| `images`      | object | Coverage for image files                           |
| `videos`      | object | Coverage for regular video files                   |
| `music`       | object | Coverage for audio (music) files                   |
| `musicVideos` | object | Coverage for music-video files                     |

### PlaylistDTO

| Field        | Type    | Description                          |
|--------------|---------|--------------------------------------|
| `id`         | long    | Playlist primary key                 |
| `name`       | string  | Playlist name (unique per user)      |
| `trackCount` | integer | Number of tracks in the playlist     |

### SmartPlaylistDTO

| Field         | Type   | Description                                              |
|---------------|--------|----------------------------------------------------------|
| `id`          | long   | Smart playlist primary key                               |
| `name`        | string | Smart playlist name (unique per user)                    |
| `definition`  | object | The saved query tree (a group node of rules)             |
| `dateUpdated` | string | When the definition or name was last changed             |

### AlbumSummaryDTO

| Field        | Type    | Description                                          |
|--------------|---------|------------------------------------------------------|
| `id`         | long    | Album primary key                                    |
| `name`       | string  | Album name, or `null`                                |
| `artist`     | string  | Album artist (informational only), or `null`         |
| `year`       | integer | Release year, or `null`                              |
| `coverId`    | long    | Cover primary key, or `null` when the album has none |
| `trackCount` | integer | Tracks grouped under the album                       |

### AlbumGroupDTO

| Field        | Type    | Description                                                   |
|--------------|---------|---------------------------------------------------------------|
| `value`      | string  | The shared group value: artist, genre, or year as text        |
| `coverId`    | long    | A randomly chosen representative cover id, or `null`          |
| `albumCount` | integer | Number of albums in the group                                 |

### MediaUploadResultDTO

| Field      | Type    | Description                                    |
|------------|---------|------------------------------------------------|
| `uploaded` | integer | Files that were stored and indexed             |
| `rejected` | integer | Files that were not stored                     |
| `items`    | array   | One `MediaUploadItemDTO` per submitted file, in submission order |

### MediaUploadItemDTO

| Field         | Type    | Description                                                            |
|---------------|---------|------------------------------------------------------------------------|
| `filename`    | string  | The original name of the submitted file                                |
| `uploaded`    | boolean | Whether the file was stored and indexed                                |
| `destination` | string  | Where it was stored, relative to its library folder root; `null` when rejected |
| `message`     | string  | Confirmation, or the reason for the rejection                          |

### ScreenshotUploadDTO

| Field         | Type   | Description                                                              |
|---------------|--------|--------------------------------------------------------------------------|
| `filename`    | string | The name the frame was stored under (may differ from the requested one)  |
| `destination` | string | Path relative to the photos library folder root, including the file name |

### MediaExportDTO

| Field                  | Type    | Description                                                     |
|------------------------|---------|-----------------------------------------------------------------|
| `kind`                 | string  | The exported library kind: `PHOTOS`, `MUSIC`, `MUSIC_VIDEOS`    |
| `exportedAt`           | string  | When the export was produced                                    |
| `count`                | integer | Number of items in the export                                   |
| `skippedWithoutHash`   | integer | Records left out because their media file has no hash yet       |
| `skippedDuplicateHash` | integer | Records left out because another item already holds the hash    |
| `items`                | object  | The exported items, keyed by content hash                       |

Each item is a `PhotoLibraryExportItemDTO` (`mediaKind`, `path`, `filename`, `dateAdded`, `users`) or
a `MusicExportItemDTO` (the same plus `songName`, `artist`, `genre`, `album`, `year`, `live`,
`source`, `coverPath`). `users` maps a username to `{ rating, views, lastViewDate, playlists }`;
`playlists` is present for music items only. Null fields are omitted from the JSON.

### RehashJobResponse

| Field              | Type    | Description                                                  |
|--------------------|---------|--------------------------------------------------------------|
| `jobId`            | string  | Unique job identifier; `null` when no rehash has run         |
| `status`           | string  | Lifecycle state (e.g. `RUNNING`, `COMPLETED`, `CANCELLED`)   |
| `startedAt`        | string  | When the rehash started; `null` for idle                     |
| `completedAt`      | string  | When it finished; `null` while running                       |
| `processed`        | integer | Media files visited so far                                   |
| `total`            | integer | Media files to visit, or `0` before they have been counted   |
| `progressPercent`  | integer | Overall progress in the range [0, 100]                       |
| `elapsedSeconds`   | long    | How long it has been running, or took                        |
| `remainingSeconds` | long    | Estimated seconds still to go; `null` when not yet estimable |
| `updated`          | integer | Files whose stored hash was out of date and has been replaced |
| `unchanged`        | integer | Files whose stored hash was already correct                  |
| `missing`          | integer | Files whose recorded path is not a regular file on disk      |
| `failed`           | integer | Files that exist but could not be read                       |
| `message`          | string  | Human-readable outcome or failure description                |

### MusicVideoThumbnailJobResponse

The same envelope as `RehashJobResponse` (`jobId`, `status`, `startedAt`, `completedAt`, `processed`,
`total`, `progressPercent`, `elapsedSeconds`, `remainingSeconds`, `message`) with its own counters.

| Field       | Type    | Description                                              |
|-------------|---------|----------------------------------------------------------|
| `generated` | integer | Music videos that were given a new thumbnail             |
| `skipped`   | integer | Music videos that already carried one when reached       |
| `missing`   | integer | Music videos whose recorded path is not a file on disk   |
| `failed`    | integer | Music videos that exist but yielded no decodable frame   |

### ImportJobResponse

The same envelope as `RehashJobResponse`, plus the library `kind` being imported into, the active
`phase`, and the counters below.

| Field                 | Type    | Description                                                   |
|-----------------------|---------|---------------------------------------------------------------|
| `matched`             | integer | Entries whose content hash matched at least one media file    |
| `notFound`            | integer | Entries whose hash matched no media file in this library      |
| `tagsUpdated`         | integer | Tag records restored (a track's metadata, or an item's date added) |
| `userTagsUpdated`     | integer | Per-user rating records created or updated                    |
| `usersNotFound`       | integer | Per-user entries skipped because no such user exists here     |
| `playlistsCreated`    | integer | Playlists created because the named user had none by that name |
| `playlistTracksAdded` | integer | Tracks added to a playlist they were not already in           |
| `coversRestored`      | integer | Album covers set or upgraded from the backup directory        |
| `coversKept`          | integer | Album covers left alone because the current one is as good    |
| `coversMissing`       | integer | Entries naming a cover not present under the covers backup    |

### LanguageDTO

| Field         | Type   | Description          |
|---------------|--------|----------------------|
| `text`        | string | Translation key      |
| `translation` | string | Translated text      |

### Paginated Response (Spring Page)

Endpoints returning a paginated list wrap results in a Spring `Page` envelope.

| Field              | Type    | Description                          |
|--------------------|---------|--------------------------------------|
| `content`          | array   | The records for this page            |
| `totalElements`    | integer | Total matching records               |
| `totalPages`       | integer | Total number of pages                |
| `number`           | integer | Current page index (zero-based)      |
| `size`             | integer | Requested page size                  |
| `numberOfElements` | integer | Records returned in this page        |
| `first`            | boolean | Whether this is the first page       |
| `last`             | boolean | Whether this is the last page        |
| `empty`            | boolean | Whether `content` is empty           |

---

## General

Base path: `/api/general`

### GET /api/general/appAlive

Health check — returns `true` when the backend is reachable.

**Request:** none

**Response:** `200 OK`

```json
true
```

---

### GET /api/general/version

Returns the application version as defined in the Maven POM (`project.version`).

**Request:** none

**Response:** `200 OK` — plain text

```
1.5.0
```

**Use case:** Display the running application version in the UI (e.g. the sidebar footer).

---

### GET /api/general/stats

Returns a snapshot of entity counts across the entire database: files (broken down by kind), tags, rated tags, total views, thumbnails, folders, library folders, and users.

**Request:** none

**Response:** `200 OK` — `DatabaseStatsResponse`

```json
{
  "totalFiles": 1573,
  "images": 1500,
  "videos": 50,
  "audioFiles": 20,
  "documents": 3,
  "tags": 1573,
  "userTags": 85,
  "rated": 42,
  "totalViews": 8350,
  "thumbnails": 1580,
  "folders": 4,
  "libraryFolders": 1,
  "users": 2
}
```

**Use case:** Display an at-a-glance summary of the library on the library folders view.

---

### DELETE /api/general/deep-clean

Deletes all library data from the database and removes the thumbnails directory from disk. **This operation is irreversible.**

**Request:** none

**Response:** `200 OK` (empty body)

**Use case:** Full reset of the library before re-importing from scratch.

---

## Languages

Base path: `/api/languages`

### GET /api/languages/all/{language}

Returns all translation entries for the given language code.

**Path parameters**

| Parameter  | Type   | Description                       |
|------------|--------|-----------------------------------|
| `language` | string | ISO language code, e.g. `en`, `el` |

**Response:** `200 OK` — array of `LanguageDTO`

```json
[
  { "text": "Settings", "translation": "Settings" },
  { "text": "Search",   "translation": "Search" }
]
```

**Errors**

| Status | Condition                          |
|--------|------------------------------------|
| 404    | Translations could not be loaded   |

**Use case:** Bootstrap the frontend i18n strings on application load.

---

### POST /api/languages

Sets the active UI language used by server-side messages.

**Request body**

```json
{ "language": "el" }
```

**Response:** `200 OK` (empty body)

**Use case:** Synchronize server-side language with the user's frontend language selection.

---

## Settings

Base path: `/api/settings`

### GET /api/settings/all

Returns all application settings.

**Request:** none

**Response:** `200 OK` — array of `Setting`

```json
[
  { "id": 1, "settingName": "defaultActionsLanguage", "settingValue": "en" },
  { "id": 2, "settingName": "thumbnailSize",          "settingValue": "200" }
]
```

---

### GET /api/settings/{id}

Returns a single setting by its primary key.

**Path parameters**

| Parameter | Type | Description        |
|-----------|------|--------------------|
| `id`      | long | Setting primary key |

**Response:** `200 OK` — `Setting`

```json
{ "id": 1, "settingName": "defaultActionsLanguage", "settingValue": "en" }
```

**Errors**

| Status | Condition              |
|--------|------------------------|
| 404    | Setting id not found   |

---

### GET /api/settings/name/{name}

Returns a single setting by its name.

**Path parameters**

| Parameter | Type   | Description   |
|-----------|--------|---------------|
| `name`    | string | Setting name  |

**Response:** `200 OK` — `Setting`

**Errors**

| Status | Condition                |
|--------|--------------------------|
| 404    | Setting name not found   |

---

### GET /api/settings/metadata/{language}

Returns the metadata for every setting, resolved for the requested language: each entry describes the
setting's human-readable label, description, input type, and any validation constraints (min/max,
allowed options), so the settings page can render and validate the form without hard-coding it.

**Path parameters**

| Parameter  | Type   | Description                                   |
|------------|--------|-----------------------------------------------|
| `language` | string | ISO language code, e.g. `en` or `el`          |

**Response:** `200 OK` — array of `SettingMetadataDTO` in definition order.

---

### PUT /api/settings

Updates the value of an existing setting. Only `settingValue` is modified.

**Request body**

| Field          | Type   | Required | Description                 |
|----------------|--------|----------|-----------------------------|
| `id`           | string | yes      | Setting primary key as string |
| `settingName`  | string | yes      | Setting name                |
| `settingValue` | string | yes      | New value                   |

```json
{
  "id": "1",
  "settingName": "defaultActionsLanguage",
  "settingValue": "el"
}
```

**Response:** `200 OK` (empty body)

**Errors**

| Status | Condition                                    |
|--------|----------------------------------------------|
| 404    | Setting id not found                         |
| 400    | New value fails field validation             |

**Use case:** Change the app language or any other configurable setting from the settings screen.

---

## Library Folders

Base path: `/api/library-folders`

Library folders are the root directories that are recursively scanned for media files.

### GET /api/library-folders

Returns all configured library folders.

**Request:** none

**Response:** `200 OK` — array of `LibraryFolder`

```json
[
  { "id": 1, "path": "/home/user/Photos" },
  { "id": 2, "path": "/mnt/nas/Pictures" }
]
```

---

### GET /api/library-folders/{id}

Returns a single library folder by its primary key.

**Path parameters**

| Parameter | Type | Description            |
|-----------|------|------------------------|
| `id`      | long | Library folder primary key |

**Response:** `200 OK` — `LibraryFolder`

**Errors**

| Status | Condition                    |
|--------|------------------------------|
| 404    | Library folder not found     |

---

### POST /api/library-folders

Creates a new library folder.

**Request body** — `LibraryFolder` (without `id`)

```json
{ "path": "/home/user/NewAlbum" }
```

**Response:** `201 Created` — persisted `LibraryFolder`

```json
{ "id": 3, "path": "/home/user/NewAlbum" }
```

**Errors**

| Status | Condition                       |
|--------|---------------------------------|
| 400    | `path` is blank or invalid      |

**Use case:** Add a new disk folder to the scan configuration before running a scan.

---

### PUT /api/library-folders/{id}

Updates the path of an existing library folder.

**Path parameters**

| Parameter | Type | Description            |
|-----------|------|------------------------|
| `id`      | long | Library folder primary key |

**Request body**

```json
{ "path": "/home/user/RenamedAlbum" }
```

**Response:** `200 OK` — updated `LibraryFolder`

**Errors**

| Status | Condition                    |
|--------|------------------------------|
| 404    | Library folder not found     |
| 400    | `path` is blank or invalid   |

---

### DELETE /api/library-folders/{id}

Deletes a library folder configuration record.

**Path parameters**

| Parameter | Type | Description            |
|-----------|------|------------------------|
| `id`      | long | Library folder primary key |

**Response:** `204 No Content`

**Errors**

| Status | Condition                    |
|--------|------------------------------|
| 404    | Library folder not found     |

---

## Background Scan

Base path: `/api/scan`

The background scan runs asynchronously across all configured library folders. Only one scan can run at a time. Poll `/api/scan/status` to track progress.

### POST /api/scan/start

Starts a background library scan. Returns immediately once the job is queued.

**Request:** none

**Response:** `202 Accepted` — initial `ScanJobResponse`

```json
{
  "jobId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "RUNNING",
  "phase": "COLLECTING",
  "startedAt": "2024-03-15T08:00:00Z",
  "completedAt": null,
  "added": 0,
  "skipped": 0,
  "errors": 0,
  "foldersScanned": 0,
  "foldersSkipped": 0,
  "tagged": 0,
  "removed": 0,
  "totalFolders": 0,
  "totalFiles": 0,
  "totalMediaFilesInLibrary": 0,
  "totalFoldersToClean": 0,
  "foldersClean": 0,
  "progressPercent": 0,
  "errorLogs": [],
  "initialFilesCount": 1200,
  "message": "Scan started"
}
```

**Errors**

| Status | Condition                     |
|--------|-------------------------------|
| 409    | A scan is already running     |

**Use case:** Trigger an index update after adding new photos to a library folder.

---

### POST /api/scan/cancel

Requests cancellation of the currently running scan. The scan stops at the next safe checkpoint; already-completed work is retained.

**Request:** none

**Response:** `200 OK` — `ScanJobResponse` reflecting the cancelling job

**Errors**

| Status | Condition                     |
|--------|-------------------------------|
| 409    | No scan is currently running  |

---

### POST /api/scan/backfill-metadata

Starts a background metadata backfill that re-reads EXIF metadata for already-tagged photos and fills
in any field the original scan left empty (GPS coordinates, camera make and model, date taken,
dimensions). Progress is reported through the same `ScanJobResponse` and `GET /api/scan/status` as a
scan.

**Request:** none

**Response:** `202 Accepted` — the initial `ScanJobResponse` for the backfill job.

**Errors:** `409` when a scan or a backfill is already running.

---

### GET /api/scan/status

Returns the current status of the most recent scan job. Returns an idle placeholder if no scan has been started since the application launched.

**Request:** none

**Response:** `200 OK` — `ScanJobResponse`

```json
{
  "jobId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "COMPLETED",
  "phase": null,
  "startedAt": "2024-03-15T08:00:00Z",
  "completedAt": "2024-03-15T08:05:32Z",
  "added": 47,
  "skipped": 1153,
  "errors": 0,
  "foldersScanned": 12,
  "foldersSkipped": 88,
  "tagged": 47,
  "removed": 2,
  "totalFolders": 100,
  "totalFiles": 47,
  "totalMediaFilesInLibrary": 1200,
  "totalFoldersToClean": 12,
  "foldersClean": 12,
  "progressPercent": 100,
  "errorLogs": [],
  "initialFilesCount": 1153,
  "message": "Scan completed"
}
```

**Use case:** Poll this endpoint every few seconds to update a progress bar in the UI during a scan.

---

## Photos

Base path: `/api/photos`

### POST /api/photos/scan

Synchronously scans a specific folder path for image files. The path must match a configured library folder or be a subfolder of one.

**Request body**

```json
{ "folderPath": "/home/user/Photos/Vacation2024" }
```

**Response:** `200 OK` — `ScanResult`

```json
{
  "added": 12,
  "skipped": 5,
  "errors": 0,
  "foldersScanned": 3,
  "foldersSkipped": 1,
  "removed": 0,
  "message": "Scan completed: 12 added, 5 skipped, 0 errors"
}
```

**Errors**

| Status | Condition                                              |
|--------|--------------------------------------------------------|
| 400    | `folderPath` is blank or does not match a library folder |

**Use case:** Quickly re-index a single folder after adding a small batch of photos.

---

### POST /api/photos/scan-library

Synchronously scans all configured library folders for image files.

**Request:** none

**Response:** `200 OK` — `ScanResult` (aggregated across all library folders)

**Use case:** Perform an initial full library import.

---

### GET /api/photos/all

Returns a paginated list of all photo media files, newest first.

**Query parameters**

| Parameter | Type    | Default | Description                  |
|-----------|---------|---------|------------------------------|
| `page`    | integer | `0`     | Zero-based page index        |
| `size`    | integer | `20`    | Number of items per page     |

**Response:** `200 OK` — paginated `MediaFile` records

```json
{
  "content": [
    {
      "id": 101,
      "libraryFolder": { "id": 1, "path": "/home/user/Photos" },
      "path": "Vacation2024",
      "filename": "beach.jpg",
      "hash": null,
      "kind": "IMAGE",
      "thumbnailId": 55
    }
  ],
  "totalElements": 1200,
  "totalPages": 60,
  "number": 0,
  "size": 20,
  "first": true,
  "last": false,
  "empty": false
}
```

**Use case:** Display the full photo library in an infinite-scroll grid.

---

### GET /api/photos/geo

Returns photos that carry GPS coordinates, as lightweight geolocated points for the map view. Only
images are considered — videos have no GPS metadata — and a photo is included only when both its
latitude and longitude are present, since a single coordinate cannot be plotted. The matching set is
returned at once so the client can cluster nearby points, grouping locations where many photos were
taken.

**Scoping precedence** (mirrors `/api/photos/batch`):
1. If `query` has active filters (text or rating) → scope to search results
2. Else if `folderId` is provided → scope to that folder's subtree
3. Else → entire library

**Query parameters**

| Parameter  | Type   | Default | Description                                                     |
|------------|--------|---------|-------------------------------------------------------------------|
| `folderId` | long   | —       | Scope to this folder's subtree; ignored when `query` is active   |
| `query`    | string | —       | JSON-encoded `PhotoQuery` object (see `/api/photos/batch`)        |

**Response:** `200 OK` — array of `PhotoLocation` objects (empty when no matching photo has coordinates)

```json
[
  {
    "id": 101,
    "filename": "beach.jpg",
    "thumbnailId": 55,
    "latitude": 40.6401,
    "longitude": 22.9444
  }
]
```

**Use case:** Plot geotagged photos on a map, clustering nearby photos into a single marker.

---

### GET /api/photos/batch

Returns a batch of photos for the slideshow, generating any missing thumbnails on the fly.

**Scoping precedence:**
1. If `query` has active filters (text or rating) → scope to search results
2. Else if `folderId` is provided → scope to that folder's subtree
3. Else → entire library

**Query parameters**

| Parameter   | Type    | Default | Description                                                       |
|-------------|---------|---------|-------------------------------------------------------------------|
| `count`     | integer | `10`    | Number of photos to return (clamped to 1–50)                     |
| `folderId`  | long    | —       | Scope to this folder's subtree; ignored when `query` is active   |
| `doShuffle` | boolean | `true`  | `true` = random subset; `false` = sequential by id              |
| `afterId`   | long    | —       | Sequential mode only: resume after this photo id                 |
| `query`     | string  | —       | JSON-encoded `PhotoQuery` object (see below)                     |

**PhotoQuery JSON structure** (passed as a URL-encoded JSON string to `query`)

```json
{ "text": "beach", "rating": 4 }
```

Both fields are optional. The query is active when `text` is non-blank or `rating` is set.

**Response:** `200 OK` — array of `MediaFile` (with `thumbnailId` populated)

```json
[
  {
    "id": 42,
    "libraryFolder": { "id": 1, "path": "/home/user/Photos" },
    "path": "Vacation2024",
    "filename": "beach.jpg",
    "hash": "abc123",
    "kind": "IMAGE",
    "thumbnailId": 7
  }
]
```

**Response:** `204 No Content` — when no photos match the requested scope

**Use case:** Drive the slideshow by repeatedly calling this endpoint. In shuffle mode each call returns a fresh random subset. In sequential mode, pass the last received photo's `id` as `afterId` to page forward through the library.

**Example — shuffle 10 photos from folder 3:**
```
GET /api/photos/batch?count=10&folderId=3&doShuffle=true
```

**Example — next 5 photos sequentially after id 99:**
```
GET /api/photos/batch?count=5&doShuffle=false&afterId=99
```

**Example — slideshow of 4-star beach photos:**
```
GET /api/photos/batch?count=20&query=%7B%22text%22%3A%22beach%22%2C%22rating%22%3A4%7D
```

---

### GET /api/photos/{id}

Returns the full detail view for a single photo.

When the photo carries GPS coordinates but has no cached `locationName` yet, this reverse-geocodes
them via `GeocodingService` (OpenStreetMap's Nominatim API) and persists the result onto the
`PhotoTag`, so the external lookup happens once per photo and every later request for it is served
from the cached value. A failed or unresolved lookup simply leaves `locationName` as `null` — it
never fails the request.

**Path parameters**

| Parameter | Type | Description         |
|-----------|------|---------------------|
| `id`      | long | Media file primary key |

**Response:** `200 OK` — `PhotoDetailDTO`

```json
{
  "id": 42,
  "path": "/home/user/Photos/Vacation2024",
  "filename": "beach.jpg",
  "hash": "abc123",
  "kind": "IMAGE",
  "name": "Sunset at the beach",
  "description": null,
  "album": "Vacation 2024",
  "filesize": 3145728,
  "width": 4032,
  "height": 3024,
  "viewCount": 5,
  "rating": 4,
  "dateTaken": "2024-07-14T18:35:00",
  "latitude": 37.9838,
  "longitude": 23.7275,
  "locationName": "Athens, Greece",
  "cameraMake": "Apple",
  "cameraModel": "iPhone 15 Pro",
  "mimeType": "image/jpeg",
  "dateCreated": "2024-07-15T10:00:00",
  "dateUpdated": "2024-07-20T09:12:00"
}
```

**Errors**

| Status | Condition             |
|--------|-----------------------|
| 404    | Photo not found       |

---

### GET /api/photos/{id}/image

Serves the raw image bytes of the specified photo.

**Path parameters**

| Parameter | Type | Description         |
|-----------|------|---------------------|
| `id`      | long | Media file primary key |

**Response:** `200 OK` — binary image body with the appropriate `Content-Type` (e.g. `image/jpeg`, `image/png`)

**Errors**

| Status | Condition                                  |
|--------|--------------------------------------------|
| 404    | Photo record not found, or file not on disk |

**Use case:** Display the full-resolution image in the slideshow or detail view.

---

### PATCH /api/photos/{id}/rating

Sets or updates the authenticated user's rating for a photo (creates a per-user tag record if none exists).

**Path parameters**

| Parameter | Type | Description         |
|-----------|------|---------------------|
| `id`      | long | Media file primary key |

**Request body**

```json
{ "rating": 4 }
```

`rating` must be an integer between 1 and 5 (inclusive).

**Response:** `200 OK` — updated `PhotoDetailDTO`

**Errors**

| Status | Condition                          |
|--------|------------------------------------|
| 404    | Photo not found                    |
| 400    | `rating` missing or out of 1–5 range |

**Use case:** Star-rating a photo from the slideshow or detail panel.

---

### POST /api/photos/{id}/view

Increments the authenticated user's view counter for a photo by one and records the last view date (creates a per-user tag record if none exists).

**Path parameters**

| Parameter | Type | Description         |
|-----------|------|---------------------|
| `id`      | long | Media file primary key |

**Request:** none

**Response:** `200 OK` — updated `PhotoDetailDTO`

**Errors**

| Status | Condition        |
|--------|------------------|
| 404    | Photo not found  |

**Use case:** Call this each time the slideshow displays a photo to track view counts.

---

### POST /api/photos/{id}/thumbnail

Generates a thumbnail for the specified photo. Returns the existing thumbnail id if one already exists.

**Path parameters**

| Parameter | Type | Description         |
|-----------|------|---------------------|
| `id`      | long | Media file primary key |

**Request:** none

**Response:** `200 OK`

```json
{ "thumbnailId": 55 }
```

**Errors**

| Status | Condition                                        |
|--------|--------------------------------------------------|
| 404    | Photo not found, or image file not on disk       |
| 400    | Thumbnail generation failed                      |

---

### DELETE /api/photos/{id}

Deletes a single photo: its database record (and its photo tag and thumbnail record), and removes the original image file and its thumbnail file from disk.

**Path parameters**

| Parameter | Type | Description         |
|-----------|------|---------------------|
| `id`      | long | Media file primary key |

**Response:** `204 No Content`

**Errors**

| Status | Condition        |
|--------|------------------|
| 404    | Photo not found  |

---

### DELETE /api/photos/all

Deletes all photo records from the database. Physical files are not affected.

**Request:** none

**Response:** `204 No Content`

**Use case:** Clear the index before a full re-scan.

---

### GET /api/photos/tags/export

Exports all per-user tag entries, across every user, that have at least one view or a rating set and
whose media file carries a content hash. Each entry identifies the file by its content hash and the
owning user by username, so the payload is portable across installations with different file paths.

**Request:** none

**Response:** `200 OK` — array of `TagExportItemDTO`

```json
[
  {
    "hash": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
    "username": "alice",
    "rating": 4,
    "views": 5
  },
  {
    "hash": "60303ae22b998861bce3b28f33eec1be758a213c86c93c076dbe9f558c11c752",
    "username": "bob",
    "rating": null,
    "views": 12
  }
]
```

**Use case:** Back up every user's ratings and view counts before migrating to a new installation.

---

### POST /api/photos/tags/import

Imports per-user tag data (ratings and view counts) from a JSON payload. Each entry resolves the
media file by its content hash and the user by username, then applies the entry's `rating` and
`views` to that user's tag for the file, creating the tag if it does not yet exist. When a hash is
shared by several media files (duplicates), the values are applied to every one of them.

**Request body** — array of `TagExportItemDTO`

```json
[
  {
    "hash": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
    "username": "alice",
    "rating": 4,
    "views": 5
  }
]
```

Only `rating` and `views` are written. Entries whose hash matches no media file, or whose username
matches no user, are counted as `notFound` and skipped silently.

**Response:** `200 OK`

```json
{
  "updated": 1,
  "notFound": 0
}
```

| Field      | Type    | Description                                                     |
|------------|---------|-----------------------------------------------------------------|
| `updated`  | integer | Number of user tag records created or updated                   |
| `notFound` | integer | Number of entries whose hash or username could not be resolved  |

**Use case:** Restore every user's ratings and view counts after re-scanning a moved or reinstalled library.

---

## Videos

Base path: `/api/videos`

Video endpoints mirror the photo endpoints for media of kind `VIDEO`. Videos are indexed by the same
scan as photos (no separate scan trigger) and surface in the Photos grid, search, and "Recent" view
alongside images, but they never play in the photo slideshow — they play in the Playlist, driven by
the batch endpoint below. Per-user ratings and view counts share the same `UserTag` storage as photos.

### GET /api/videos/all

Returns a paginated list of all video media files, newest first.

**Query parameters**

| Parameter | Type | Default | Description              |
|-----------|------|---------|--------------------------|
| `page`    | int  | 0       | Zero-based page index    |
| `size`    | int  | 20      | Page size                |

**Response:** `200 OK` — paginated `MediaFile` page (kind `VIDEO`).

---

### GET /api/videos/batch

Returns up to `count` videos for the Playlist, generating any missing frame thumbnails first. Scoping
precedence, shuffle/sequential behaviour, and parameters are identical to `GET /api/photos/batch`, but
the selection is restricted to kind `VIDEO`.

**Query parameters**

| Parameter   | Type    | Default | Description                                                        |
|-------------|---------|---------|--------------------------------------------------------------------|
| `count`     | int     | 10      | Number of videos to return (clamped to 1–50)                       |
| `folderId`  | long    | —       | Scope to a folder subtree; ignored when `query` is active          |
| `doShuffle` | boolean | true    | Random subset when true, sequential when false                     |
| `afterId`   | long    | —       | Sequential mode: resume after this id                              |
| `query`     | string  | —       | JSON-encoded `PhotoQuery` to scope the selection to a search       |

**Response:** `200 OK` — array of `MediaFile` (kind `VIDEO`), or `204 No Content` when none match.

---

### GET /api/videos/{id}

Returns the detail view for a single video, combining its `MediaFile`, optional `VideoTag`, and the
current user's `UserTag`.

**Response:** `200 OK` — `VideoDetailDTO`. **Errors:** `404` when the video is not found.

---

### GET /api/videos/{id}/stream

Streams the raw video bytes with HTTP range support so the browser's player can seek. With a `Range`
header the response is `206 Partial Content` carrying up to 1 MiB from the requested offset; without
one the whole file is returned as `200 OK` with `Accept-Ranges: bytes`. When the source file is gone
from disk the orphaned record is purged (record, video tag, user tags, thumbnail) before the `404`,
mirroring `GET /api/photos/{id}/image`.

**Errors:** `404` when the video record or the file on disk is missing.

---

### PATCH /api/videos/{id}/rating

Sets the current user's rating (1–5) for the video, creating a `UserTag` if none exists.

**Request body:** `{ "rating": 4 }` · **Response:** `200 OK` — `VideoDetailDTO`.

**Errors:** `400` when `rating` is missing or out of range; `404` when the video is not found.

---

### POST /api/videos/{id}/view

Increments the current user's view counter for the video by one.

**Response:** `200 OK` — `VideoDetailDTO`. **Errors:** `404` when the video is not found.

---

### POST /api/videos/{id}/thumbnail

Generates a frame thumbnail (grabbed from the middle of the video) and returns its id, or the existing
id when one already exists.

**Response:** `200 OK` — `{ "thumbnailId": 123 }`.

**Errors:** `404` when the video record or file is missing; `500` when no frame can be decoded.

---

### DELETE /api/videos/{id}

Deletes a single video: its database record (and its video tag, user tags, and thumbnail record), and
removes the original file and its thumbnail file from disk.

**Response:** `204 No Content`. **Errors:** `404` when the video is not found.

---

### DELETE /api/videos/all

Deletes all video records from the database. Physical files are not affected.

**Response:** `204 No Content`. **Use case:** Clear the video index before a full re-scan.

---

## Music

Base path: `/api/music`

Music endpoints serve audio tracks and music videos — files tagged with a `MusicTag` because they
live in a `MUSIC` or `MUSIC_VIDEOS` library folder. They never appear in the photo slideshow or the
video playlist; they play only in the Music view. Per-user ratings and play counts share the same
`UserTag` storage as photos and videos. Every response body is a `MusicDetailDTO` (see
[Common Schemas](#common-schemas)) unless noted otherwise.

### GET /api/music/all

Returns a paginated list of tracks and music videos, newest first, each combined with the current
user's rating and play count.

**Query parameters**

| Parameter  | Type   | Default | Description                                                    |
|------------|--------|---------|----------------------------------------------------------------|
| `page`     | int    | 0       | Zero-based page index                                          |
| `size`     | int    | 50      | Page size                                                      |
| `kind`     | string | —       | Restrict to `MUSIC` or `MUSIC_VIDEO`; omit for both            |
| `source`   | string | —       | Restrict to one `MusicSource`: `REGULAR`, `LOSSLESS`, `VINYL`, `CDRIP`. `REGULAR` also matches legacy rows whose column is null |
| `songName` | string | —       | Exact title match (case-insensitive), as produced by clicking a Title cell |
| `artist`   | string | —       | Exact artist match (case-insensitive)                          |
| `album`    | string | —       | Exact album name match (case-insensitive); matches through the track's `Album`, so a track with no album never matches |
| `genre`    | string | —       | Exact genre match (case-insensitive)                           |
| `year`     | int    | —       | Album release year; matches through the track's `Album`        |
| `albumId`  | long   | —       | Only tracks linked to this album, used when opening one album from the album panes (matches by id, so two albums sharing a name stay distinct) |
| `rating`   | int    | —       | Rating the current user's `UserTag` must satisfy (0–5; `0` is the "unrated" bucket and ignores `ratingOp`) |
| `ratingOp` | string | `eq`    | Rating comparison: `eq`, `gt`, `lt`, `gte`, `lte`              |
| `sort`     | string | —       | Column to order by: `title`, `artist`, `album`, `genre`, `year`, `rating`, `plays`, `dateAdded` |
| `dir`      | string | `desc`  | Sort direction: `asc` or `desc`                                |

**Ordering.** With **no** `sort` the list is clustered by album — an album is positioned by its
representative track and moves as one block, so its tracks never interleave with another's — and
falls back to date added descending. With a `sort` the chosen column wins and the clustering is
dropped, remaining only as a tiebreaker (album key, then sleeve position, then media file id
descending).

**Response:** `200 OK` — paginated `MusicDetailDTO`.

---

### GET /api/music/albums

Returns a page of album summaries, one per album holding at least one track, each with its cover and
track count — the data source for both the albums grid and the coverflow album slider. Albums are
ordered case-insensitively by name; albums with no tracks never appear.

**Query parameters**

| Parameter | Type   | Default | Description                                                    |
|-----------|--------|---------|----------------------------------------------------------------|
| `page`    | int    | 0       | Zero-based page index                                          |
| `size`    | int    | 48      | Albums per page                                                |
| `artist`  | string | —       | Restrict to albums by this artist (case-insensitive)           |
| `genre`   | string | —       | Restrict to albums holding at least one track of this genre (their full track count is still reported) |
| `year`    | int    | —       | Restrict to albums released in this year                       |

At most one of `artist`, `genre`, and `year` is honoured, in that order of precedence; they back
drilling into one group tile.

**Response:** `200 OK` — paginated `AlbumSummaryDTO`.

---

### GET /api/music/album-groups

Returns the album groups for one dimension, so the album panes can offer an artist / genre / year
overview: one tile per group with its album count and a randomly chosen representative cover. Artist
and year are read from the `Album`, genre from each track, so an album whose tracks span genres
counts under each. Artists and genres are ordered case-insensitively by name, years descending.

**Query parameters**

| Parameter | Type   | Default | Description                                    |
|-----------|--------|---------|------------------------------------------------|
| `by`      | string | —       | Grouping dimension: `artist`, `genre`, or `year` (required) |

**Response:** `200 OK` — array of `AlbumGroupDTO`. **Errors:** `400` when `by` is not a known
dimension.

---

### GET /api/music/random

Returns one track chosen uniformly at random from the whole library, so shuffle playback can draw
from every page rather than only the tracks loaded in the client. The kind, source, and rating
filters are honoured so shuffle draws from the same pool the toolbar defines; the per-field filters
are not.

**Query parameters**

| Parameter   | Type   | Default | Description                                                    |
|-------------|--------|---------|----------------------------------------------------------------|
| `kind`      | string | —       | Restrict the random pick to `MUSIC` or `MUSIC_VIDEO`           |
| `source`    | string | —       | Restrict the random pick to one `MusicSource`                  |
| `excludeId` | long   | —       | Media file id to exclude (the track already playing)          |
| `rating`    | int    | —       | Rating the current user's tag must satisfy (0–5; `0` is unrated) |
| `ratingOp`  | string | `eq`    | Rating comparison: `eq`, `gt`, `lt`, `gte`, `lte`              |

**Response:** `200 OK` — `MusicDetailDTO`. **Errors:** `404` when no eligible track remains (empty
library, or a filter matching only the excluded track).

---

### GET /api/music/{id}

Returns the detail for a single track, combining its `MusicTag`, album, cover, and the current user's
`UserTag`.

**Response:** `200 OK` — `MusicDetailDTO`. **Errors:** `404` when the media file is not found or carries
no music tag.

---

### GET /api/music/{id}/page

Returns the zero-based page index the track appears on in the paginated list, so the Music view's
"current song" button can jump to the playing track after paging away. The same optional filters,
sort column, and page size as `GET /api/music/all` apply (`size`, `kind`, `source`, `songName`,
`artist`, `album`, `genre`, `year`, `albumId`, `rating`, `ratingOp`, `sort`, `dir`) and the position
is computed under exactly the same order.

**Response:** `200 OK` — `{ "page": 3 }`.

**Errors:** `404` when the track does not exist or falls outside the active filters, so it has no row
in the list.

---

### GET /api/music/{id}/stream

Streams the raw bytes (audio or music video) with HTTP range support so the browser's player can seek.
The response is **always** `206 Partial Content` carrying at most 1 MiB, even when the request has no
`Range` header — a range-less request is treated as an implicit request for the first chunk. Serving a
multi-gigabyte music video in one unbounded `200 OK` is what crashed low-memory media stacks (observed
on Xbox Edge), since some players issue a plain GET on load or after a seek.

**Errors:** `404` when the media record or the file on disk is missing.

---

### GET /api/music/covers/{coverId}

Serves an album cover image as a JPEG.

**Response:** `200 OK` — binary JPEG body with `Content-Type: image/jpeg`. **Errors:** `404` when the
cover record or its file on disk is missing.

---

### POST /api/music/albums/{albumId}/cover

Sets an album's cover from a hand-picked image, replacing whatever cover the album had — the most
permissive of the three attach modes (the scan never replaces, a rescan only upgrades a
lower-resolution cover, this always wins). The cover hangs off the `Album`, so one selected track sets
it for every track grouped under it; music videos have no album and can never carry one.

The bytes are re-encoded as JPEG and stored as a **new** `Cover` row with its own id, so no client is
served a stale cached image, and the superseded row and file are deleted.

Then, and **only when the `editTagsOnFiles` setting is enabled** (it defaults to off), the image is
embedded as the front-cover artwork of **every** audio track of the album — not only the selected
ones — and each rewritten file is queued for a deferred hash refresh. With the setting off,
`taggedFiles` is `0` and the files are left byte-for-byte untouched.

**Request:** `multipart/form-data` with a single `file` part (an image, max 20 MB).

**Response:** `200 OK`

```json
{ "albumId": 7, "coverId": 913, "taggedFiles": 12 }
```

**Errors:** `400` when no file was sent, it is not an image, it exceeds 20 MB, or it cannot be
decoded or written; `404` when no album has that id.

---

### GET /api/music/albums/{albumId}/cover-candidates

Searches the online providers for album artwork, so the edit modal can offer covers for an album the
scan found no embedded picture on. Two providers are consulted in order: **MusicBrainz + the Cover Art
Archive** first (its images are contributed for reuse), and the **iTunes Search API** only when the
archive returns nothing (far wider coverage of commercial releases). Neither needs an API key.

Nothing is stored and no match is applied automatically — matching an album by name is guesswork, so
the candidates come back for the user to pick from. A lookup happens only on this request; nothing in
the application contacts a provider on its own.

Each candidate also carries the pixel size of its full-size image, since that is what tells two
candidates of the same artwork apart. Neither provider reports it, so the server measures it by
reading each image's header — a few kilobytes, not a download — over the same host allowlist the
download below enforces. `width` and `height` are `null` when an image could not be measured.

| Param    | Type   | Default            | Description                                                     |
| -------- | ------ | ------------------ | --------------------------------------------------------------- |
| `artist` | string | the album's artist | Artist to search for, so unsaved edit-field text can be searched |
| `album`  | string | the album's name   | Album name to search for; format/edition brackets such as `(flac)` or `[Deluxe]` are stripped before searching |

**Response:** `200 OK` — a possibly empty array of candidates.

```json
[
  {
    "source": "Cover Art Archive",
    "title": "…Nothing Like the Sun",
    "artist": "Sting",
    "year": 1987,
    "thumbnailUrl": "https://coverartarchive.org/release/6f84…/1156386881-250.jpg",
    "imageUrl": "https://coverartarchive.org/release/6f84…/1156386881-500.jpg",
    "width": 500,
    "height": 500
  }
]
```

**Errors:** `400` when neither the request nor the album carries an album name to search for;
`404` when no album has that id.

---

### POST /api/music/albums/{albumId}/cover-from-url

Sets an album's cover from a candidate the user picked out of a lookup. The server downloads the image
itself and then stores it exactly as a hand-picked upload is stored — new `Cover` row, superseded one
deleted, and the artwork embedded into the album's files when `editTagsOnFiles` is on — so the two
paths differ only in where the bytes came from.

The URL is checked against an allowlist of the providers' own hosts (`coverartarchive.org`,
`archive.org`, `mzstatic.com`, and their subdomains), HTTPS only, and **every redirect is re-checked**
against the same list. It is therefore not a way to make the application fetch an address of the
caller's choosing.

**Request body**

```json
{ "url": "https://coverartarchive.org/release/6f84…/1156386881-500.jpg" }
```

**Response:** `200 OK` — the same body as the cover upload.

```json
{ "albumId": 7, "coverId": 914, "taggedFiles": 12 }
```

**Errors:** `400` when `url` is missing, is not a supported provider address, could not be downloaded,
or is not a storable image; `404` when no album has that id.

---

### POST /api/music/albums/merge

Merges several albums into one, putting back together a release the scan split across separate album
rows — album identity is the (path, name) folder key, so tracks downloaded into different folders, or
whose tags spell the album name with different casing or punctuation, each resolve to an album of
their own and show as several tiles of the same record in the album panes.

Every track of the other albums is reassigned to the album kept (its name is written onto each moved
track's own tag as well), and the rows they leave empty are deleted. The kept album keeps its own
cover and adopts one from a merged-away album only when it has none; every other cover is deleted
with its row rather than left as an orphan.

This is a **database regrouping only**: no file is moved, renamed, or retagged.

**Request body**

```json
{ "ids": [7, 41, 58], "targetId": 7 }
```

| Field      | Type   | Default | Description                                                          |
| ---------- | ------ | ------- | -------------------------------------------------------------------- |
| `ids`      | long[] | —       | The albums to merge; at least two distinct ids                        |
| `targetId` | long   | —       | Which of `ids` to keep; omitted, the album holding the most tracks survives (ties broken by the lowest id) |

**Response:** `200 OK`

```json
{ "targetId": 7, "targetName": "...Nothing Like the Sun", "mergedAlbums": 2, "movedTracks": 6, "trackCount": 11 }
```

**Errors:** `400` when `ids` holds fewer than two distinct ids or `targetId` is not one of them;
`404` when any id has no album row.

---

### PATCH /api/music/{id}/rating

Sets the current user's rating (1–5) for the track, creating a `UserTag` if none exists.

**Request body:** `{ "rating": 4 }` · **Response:** `200 OK` — `MusicDetailDTO`.

**Errors:** `400` when `rating` is missing or out of range; `404` when the track is not found.

---

### PATCH /api/music/{id}/live

Sets whether the track is a live recording, correcting a scan misclassification.

**Request body:** `{ "live": true }` · **Response:** `200 OK` — `MusicDetailDTO`.

**Errors:** `400` when `live` is missing; `404` when the track is not found.

---

### PATCH /api/music/{id}

Updates editable metadata the scan read incorrectly (or could not read). Any of `songName`, `artist`,
`genre`, `source` (stored on the track's `MusicTag`), `album`, and `year` may be present; absent fields
are unchanged and a blank value clears the field. `album` and `year` are stored on the track's own tag
and, when it has one, additionally on the shared `Album` row, so the change is visible on every track
grouped under it — a track with no album (most notably a music video) keeps them on its tag alone
rather than silently dropping them.

Alongside the database update, and **only when the `editTagsOnFiles` setting is enabled** (it defaults
to off), the same fields are written back onto the file's own embedded tags (`songName`→title,
`artist`→artist, `genre`→genre, `album`→album, `year`→year) so a later rescan or any external tool sees
the correction; a blank value deletes the tag from the file. The app-only `live` flag and the `source`
medium have no standard tag and are never written. With the setting off the file is left untouched and
only the database changes.

Both audio tracks and music videos are written — audio through the high-level jaudiotagger API, music
videos through the low-level MP4 `ilst` atom — and the write is best-effort: an unsupported container
(WAV, WebM/MKV/AVI), a missing file, or a corrupt tag is logged as a `WARN` and skipped without failing
the request. A write that **did** commit changes the file's bytes, so the file is enqueued for a
deferred hash refresh (and a tag-driven rename) rather than being re-read on this request.

**Request body** (all fields optional)

```json
{ "songName": "Title", "artist": "Artist", "genre": "Rock", "album": "Album", "year": 2024, "source": "VINYL" }
```

**Response:** `200 OK` — `MusicDetailDTO`. **Errors:** `400` when `source` does not name a known
`MusicSource`; `404` when the track is not found.

---

### PATCH /api/music/batch

Applies the same editable metadata fields as `PATCH /api/music/{id}` to many tracks at once, backing the
Music view's multi-select edit. The body carries the target ids under `ids` and the fields to apply under
`fields` (any of `songName`, `artist`, `genre`, `album`, `year`, `source`, `live`); every field follows
the single-track semantics (absent = unchanged, blank = cleared), and the same best-effort file-tag write
and deferred hash refresh happen for each track — again only when the `editTagsOnFiles` setting is
enabled.

The optional boolean `setSongsInAlbum` first pulls every selected track into the single `Album` that
the plurality of them already belong to, before the field edits apply, unifying tracks the scan split
across folders (or that carry no album at all). It is a database regrouping only: the tracks'
`album_id` is reassigned, but their files' embedded album tags are not rewritten — type the album name
into `album` to also change that on disk.

**Request body**

```json
{ "ids": [12, 34, 56], "fields": { "artist": "Unified Artist", "genre": "Jazz" }, "setSongsInAlbum": true }
```

**Response:** `200 OK` — array of `MusicDetailDTO`, one per id in the order given. **Errors:** `400` when
`ids` is missing or empty or `fields` is missing; `404` when any id has no media file or no music tag.

---

### DELETE /api/music/batch

Permanently deletes the selected tracks, removing both their database records and their files on disk,
so the Music view's multi-select delete can clear tracks from the library outright. Each track's
dependent rows (every user's tags, play-queue entries, playlist entries, and hash-refresh queue entry),
its `MusicTag`, and its `MediaFile` are removed, and its original file and any music-video thumbnail
are deleted from disk best-effort. Album and cover rows are left in place.

**Request body**

```json
{ "ids": [12, 34, 56] }
```

**Response:** `200 OK` — `{ "deleted": 3 }`. **Errors:** `400` when `ids` is missing or empty; `404`
when any id has no media file or no music tag.

---

### POST /api/music/{id}/view

Increments the current user's play counter for the track by one and records the last-play timestamp.

**Response:** `200 OK` — `MusicDetailDTO`. **Errors:** `404` when the track is not found.

---

### Play queue

A per-user, server-side ordered list of tracks lined up to play next. It overrides both sequence and
shuffle order: while the queue holds tracks the client plays the head and removes it, and only once it
is empty does playback fall back to the normal order. A track can appear in the queue at most once, so
adding a queued track is idempotent. All four endpoints return the resulting queue as an ordered array
of `MusicDetailDTO` (head first).

#### GET /api/music/queue

Returns the current user's queue in play order.

**Response:** `200 OK` — array of `MusicDetailDTO` (empty when nothing is queued).

#### POST /api/music/queue/{id}

Appends a track to the end of the queue. A track already queued is left in place.

**Response:** `200 OK` — the updated queue. **Errors:** `404` when the media file is not found or carries
no music tag.

#### DELETE /api/music/queue/{id}

Removes a track from the queue. Removing the head is how the client advances to the next queued track;
removing a track that is not queued is a no-op.

**Response:** `200 OK` — the updated queue. **Errors:** `404` when the media file is not found.

#### DELETE /api/music/queue

Clears the current user's queue entirely.

**Response:** `200 OK` — the now-empty queue (`[]`).

---

## Playlists

Base path: `/api/playlists`

A playlist is a named, ordered collection of tracks owned by one user. Unlike the play queue, it is
persistent and curated. Everything here is **per-user**: every playlist is resolved by both its id and
the calling user, so one user can never read or mutate another's — a mismatch answers `404`. Because
these are the caller's own data, every method is allowed for any authenticated user, not only `ADMIN`.

A name must be unique (case-insensitively) among that user's playlists, and a track can appear in a
playlist at most once. New tracks are appended at the end.

### GET /api/playlists

Lists the current user's playlists, each with its track count, ordered case-insensitively by name.
Empty playlists are included.

**Response:** `200 OK` — array of `PlaylistDTO`.

---

### POST /api/playlists

Creates a new, empty playlist.

**Request body:** `{ "name": "Road trip" }` · **Response:** `200 OK` — the created `PlaylistDTO`
(track count `0`).

**Errors:** `400` when the name is blank or already used by one of the user's playlists.

---

### PATCH /api/playlists/{id}

Renames one of the current user's playlists.

**Request body:** `{ "name": "New name" }` · **Response:** `200 OK` — the updated `PlaylistDTO`.

**Errors:** `400` when the name is blank or duplicate; `404` when the user has no playlist with that id.

---

### DELETE /api/playlists/{id}

Deletes a playlist together with all of its entries.

**Response:** `204 No Content`. **Errors:** `404` when the user has no playlist with that id.

---

### GET /api/playlists/{id}/tracks

Returns one page of the playlist's tracks, each combined with the user's rating and play count, so a
long playlist is paged exactly like the library list.

**Query parameters**

| Parameter | Type   | Default | Description                                                    |
|-----------|--------|---------|----------------------------------------------------------------|
| `page`    | int    | 0       | Zero-based page index                                          |
| `size`    | int    | 500     | Tracks per page                                                |
| `sort`    | string | —       | Column to order by: `title`, `artist`, `album`, `genre`, `year`, `rating`, `plays`, `dateAdded`; omit for playlist order |
| `dir`     | string | `desc`  | Sort direction, applied only with a `sort`                     |

Without a `sort` the page is read in playlist order (the order entries were appended in). With one,
the **whole** playlist is ordered by that column server-side, so the order spans every page. An entry
whose media file has lost its music tag is skipped rather than failing the request, which in playlist
order can make a page shorter than the page size while the reported total still counts the entry.

**Response:** `200 OK` — paginated `MusicDetailDTO`. **Errors:** `404` when the user has no playlist
with that id.

---

### GET /api/playlists/{id}/random

Returns a random track of the playlist, so the player can shuffle within a loaded playlist without
holding all of its tracks — the loaded page is only a slice, so the pick is made server-side over
every entry. A playlist of one track answers with that track even when it is the excluded one.

**Query parameters**

| Parameter | Type | Default | Description                                              |
|-----------|------|---------|----------------------------------------------------------|
| `exclude` | long | —       | Media file id to avoid picking (the track already playing) |

**Response:** `200 OK` — `MusicDetailDTO`. **Errors:** `404` when the user has no playlist with that
id, or it holds no playable track.

---

### POST /api/playlists/{id}/tracks/{trackId}

Adds a track to the playlist, appended at the end. A track already present is left in place, so the
call is idempotent.

**Response:** `200 OK` — the updated `PlaylistDTO` with its new track count. **Errors:** `404` when the
user has no playlist with that id, or no media file has that track id.

---

### DELETE /api/playlists/{id}/tracks/{trackId}

Removes a track from the playlist. Removing a track that is not in it is a no-op.

**Response:** `200 OK` — the updated `PlaylistDTO`. **Errors:** `404` when the user has no playlist
with that id, or no media file has that track id.

---

## Smart Playlists

Base path: `/api/smart-playlists`

A smart playlist is a **saved query** rather than a stored list of tracks: it holds a name and a
`definition` — a tree of rule groups over the music fields — that is translated into a JPA
specification and run on demand. Like ordinary playlists these are per-user data, scoped to the caller
by the controller, so every method (including the search `POST`, which is really a read) is allowed
for any authenticated user.

### GET /api/smart-playlists

Lists the current user's smart playlists with their definitions, ordered case-insensitively by name.

**Response:** `200 OK` — array of `SmartPlaylistDTO`.

---

### GET /api/smart-playlists/{id}

Returns one smart playlist, including its query definition.

**Response:** `200 OK` — `SmartPlaylistDTO`. **Errors:** `404` when the user has no smart playlist
with that id.

---

### POST /api/smart-playlists

Creates a smart playlist.

**Request body**

```json
{ "name": "Loud 90s", "definition": { "match": "all", "rules": [ { "field": "year", "op": "gte", "value": 1990 } ] } }
```

**Response:** `200 OK` — the created `SmartPlaylistDTO`. **Errors:** `400` when the name is blank or
duplicate, or the definition is malformed.

---

### PATCH /api/smart-playlists/{id}

Updates a smart playlist. A field absent from the body is left unchanged, so the same endpoint renames,
edits the definition, or both.

**Request body:** `{ "name": "New name" }`, `{ "definition": { … } }`, or both.

**Response:** `200 OK` — the updated `SmartPlaylistDTO`. **Errors:** `400` when the new name is blank
or duplicate, or the definition is malformed; `404` when the user has no smart playlist with that id.

---

### DELETE /api/smart-playlists/{id}

Deletes a smart playlist.

**Response:** `204 No Content`. **Errors:** `404` when the user has no smart playlist with that id.

---

### POST /api/smart-playlists/search

Runs a query tree against the music library and returns a page of matching tracks, so the advanced
search builder can preview a query and a saved smart playlist can be loaded into the track list and
played. An absent, null, or empty (`{}`) body matches every track.

The Music view's sibling toolbar filters are applied **on top of** the query (logical AND) when
supplied, so a smart playlist run while a kind or source is selected is narrowed to it, exactly as the
ordinary track list is.

**Query parameters**

| Parameter  | Type   | Default     | Description                                            |
|------------|--------|-------------|--------------------------------------------------------|
| `page`     | int    | 0           | Zero-based page index                                  |
| `size`     | int    | 50          | Tracks per page                                        |
| `kind`     | string | —           | Additionally restrict to `MUSIC` or `MUSIC_VIDEO`      |
| `source`   | string | —           | Additionally restrict to one `MusicSource`             |
| `rating`   | int    | —           | Additionally restrict by the current user's rating (0–5; `0` is unrated) |
| `ratingOp` | string | `eq`        | Rating comparison: `eq`, `gt`, `lt`, `gte`, `lte`      |
| `sort`     | string | `dateAdded` | Column to order by                                     |
| `dir`      | string | `desc`      | Sort direction: `asc` or `desc`                        |

**Request body:** the query tree (a group node), or omitted for "match everything".

**Response:** `200 OK` — paginated `MusicDetailDTO`. **Errors:** `400` when the query tree is
malformed, too deep, too large, or names an unknown field or operator.

---

## Folders

Base path: `/api/folders`

Folders represent the scanned subdirectory structure within the library. They are populated automatically during a scan.

### GET /api/folders

Returns all scanned folder records.

**Request:** none

**Response:** `200 OK` — array of `Folder`

```json
[
  {
    "id": 1,
    "libraryFolder": { "id": 1, "path": "/home/user/Photos" },
    "path": "Vacation2024",
    "hash": "d41d8cd9",
    "level": 1,
    "finished": true,
    "lastUpdate": "2024-07-15T10:00:00",
    "thumbnailId": 3
  }
]
```

---

### GET /api/folders/level/{level}

Returns all folders at the specified nesting level.

**Path parameters**

| Parameter | Type    | Description                                   |
|-----------|---------|-----------------------------------------------|
| `level`   | integer | Nesting level (1 = direct children of library root) |

**Response:** `200 OK` — array of `Folder`

**Use case:** Populate the top-level folder grid on the library home screen.

---

### GET /api/folders/{id}/children

Returns the direct child folders of the specified folder.

**Path parameters**

| Parameter | Type | Description         |
|-----------|------|---------------------|
| `id`      | long | Folder primary key  |

**Response:** `200 OK` — array of `Folder`

**Errors**

| Status | Condition          |
|--------|--------------------|
| 404    | Folder not found   |

**Use case:** Drill down into a folder when the user taps a folder card.

---

### GET /api/folders/{id}/photos

Returns a paginated list of image files directly inside the specified folder (not subfolders).

**Path parameters**

| Parameter | Type | Description        |
|-----------|------|--------------------|
| `id`      | long | Folder primary key |

**Query parameters**

| Parameter   | Type    | Default    | Description                                                        |
|-------------|---------|------------|--------------------------------------------------------------------|
| `page`      | integer | `0`        | Zero-based page index                                              |
| `size`      | integer | `50`       | Number of records per page                                         |
| `sortBy`    | string  | `filename` | Sort field: `filename`, `rating`, `dateTaken`, etc.               |
| `direction` | string  | `asc`      | Sort direction: `asc` or `desc`                                    |

**Response:** `200 OK` — paginated `MediaFile`

**Errors**

| Status | Condition          |
|--------|--------------------|
| 404    | Folder not found   |

**Use case:** Show the photos inside a folder in the photo grid view.

---

### GET /api/folders/by-photo/{photoId}

Returns the ancestor-to-target folder chain for the folder that directly contains the given photo. The list is ordered from the top-level ancestor down to the immediate parent folder.

**Path parameters**

| Parameter | Type | Description            |
|-----------|------|------------------------|
| `photoId` | long | Media file primary key |

**Response:** `200 OK` — ordered array of `Folder` (empty when the photo is in the library root)

```json
[
  { "id": 1, "path": "2024", "level": 1, ... },
  { "id": 5, "path": "2024/Vacation", "level": 2, ... }
]
```

**Errors**

| Status | Condition          |
|--------|--------------------|
| 404    | Photo not found    |

**Use case:** When opening a photo's detail view, navigate the breadcrumb back to the containing folder.

---

### GET /api/folders/{id}/chain

Returns the ancestor-to-target folder chain for the given folder, from the top-level ancestor down to and including the folder itself.

**Path parameters**

| Parameter | Type | Description        |
|-----------|------|--------------------|
| `id`      | long | Folder primary key |

**Response:** `200 OK` — ordered array of `Folder`

**Errors**

| Status | Condition          |
|--------|--------------------|
| 404    | Folder not found   |

**Use case:** Rebuild the breadcrumb navigation when opening a deep folder directly via a URL.

---

### POST /api/folders/{id}/thumbnail

Generates a representative thumbnail for the specified folder. Returns the existing thumbnail id if one already exists.

**Path parameters**

| Parameter | Type | Description        |
|-----------|------|--------------------|
| `id`      | long | Folder primary key |

**Request:** none

**Response:** `200 OK`

```json
{ "thumbnailId": 3 }
```

**Errors**

| Status | Condition                                            |
|--------|------------------------------------------------------|
| 404    | Folder not found, or folder contains no images       |
| 400    | Thumbnail generation failed                          |

---

### DELETE /api/folders/{id}

Deletes the folder and everything it contains: every media file in the folder and its nested
subfolders (database records, photo tags, and thumbnail records), every folder record in the
subtree, and the folder's directory tree on disk. The original files and their thumbnail files
are removed from disk as well. Sibling folders outside the subtree are unaffected.

**Path parameters**

| Parameter | Type | Description        |
|-----------|------|--------------------|
| `id`      | long | Folder primary key |

**Request:** none

**Response:** `204 No Content`

**Errors**

| Status | Condition         |
|--------|-------------------|
| 404    | Folder not found  |

**Use case:** Remove a folder and all of its images from the library in one action (e.g. bulk
delete of selected folders in edit mode).

---

### DELETE /api/folders

Deletes all folder records from the database.

**Request:** none

**Response:** `204 No Content`

**Use case:** Clear the folder index before a full re-scan.

---

## Search

Base path: `/api/search`

### GET /api/search/folders

Searches folders whose relative path contains the given query, case-insensitively. A blank query returns an empty list. Results are capped at 200.

**Query parameters**

| Parameter | Type   | Default | Description                        |
|-----------|--------|---------|------------------------------------|
| `query`   | string | `""`    | Free-text query matched against folder paths |

**Response:** `200 OK` — array of `Folder` ordered by path ascending

```json
[
  {
    "id": 5,
    "libraryFolder": { "id": 1, "path": "/home/user/Photos" },
    "path": "2024/Vacation",
    "level": 2,
    "finished": true,
    "thumbnailId": 8
  }
]
```

**Use case:** Populate the folder results section of the search view.

---

### GET /api/search/photos

Searches image files whose path or filename contains the given query, optionally filtered to an exact rating. Supports sorting and pagination for infinite scroll.

**Query parameters**

| Parameter   | Type    | Default    | Description                                                     |
|-------------|---------|------------|-----------------------------------------------------------------|
| `query`     | string  | `""`       | Free-text query matched against photo paths and filenames       |
| `rating`    | integer | —          | Exact rating filter 1–5; omit to match all ratings             |
| `page`      | integer | `0`        | Zero-based page index                                           |
| `size`      | integer | `50`       | Number of records per page                                      |
| `sortBy`    | string  | `filename` | Sort field: `filename`, `rating`, `dateTaken`, etc.            |
| `direction` | string  | `asc`      | Sort direction: `asc` or `desc`                                 |

**Response:** `200 OK` — paginated `MediaFile`

```json
{
  "content": [
    {
      "id": 42,
      "path": "Vacation2024",
      "filename": "beach.jpg",
      "thumbnailId": 7,
      ...
    }
  ],
  "totalElements": 8,
  "totalPages": 1,
  "number": 0,
  "size": 50,
  "first": true,
  "last": true,
  "empty": false
}
```

**Use case examples:**

- Search photos by filename: `GET /api/search/photos?query=beach`
- List all 5-star photos: `GET /api/search/photos?rating=5`
- 4-star photos sorted by date descending: `GET /api/search/photos?rating=4&sortBy=dateTaken&direction=desc`
- Paginated search: `GET /api/search/photos?query=sunset&page=1&size=20`

---

### GET /api/search/recent

Returns the most recently added image and video files, newest first — the data behind the "Recent"
view.

"Recently added" is derived from the primary key: media file ids come from an identity column, so a
higher id always denotes a later insertion. Ordering by id descending is an accurate proxy for date
added while being served by the existing `(kind, id)` index as an index-ordered scan that stops after
one page, which is what keeps the view fast regardless of library size.

**Query parameters**

| Parameter | Type | Default | Description           |
|-----------|------|---------|-----------------------|
| `page`    | int  | 0       | Zero-based page index |
| `size`    | int  | 50      | Page size             |

**Response:** `200 OK` — paginated `MediaFile` of kind `IMAGE` or `VIDEO`, ordered by id descending.

---

## Thumbnails

Base path: `/api/thumbnails`

### GET /api/thumbnails/{id}

Serves the generated thumbnail image as a JPEG.

**Path parameters**

| Parameter | Type | Description            |
|-----------|------|------------------------|
| `id`      | long | Thumbnail primary key  |

**Response:** `200 OK` — binary JPEG body with `Content-Type: image/jpeg`

**Errors**

| Status | Condition                                        |
|--------|--------------------------------------------------|
| 404    | Thumbnail record not found, or file not on disk  |

The 404 is a **bare** response, not the usual error envelope, and is not written to the application
log: a thumbnail is a regenerable cache artifact, and a page still holding an id the cleanup has just
deleted would otherwise produce one admin-visible error row per broken image tag. The listing
endpoints clear a media file's reference to a thumbnail record that no longer exists, so normal
on-demand generation takes over on the next view.

**Use case:** Display folder and photo thumbnails in the library grid by embedding this URL in an `<img>` tag, e.g. `/api/thumbnails/55`.

---

### DELETE /api/thumbnails/files

Deletes every **file** thumbnail — both the database records and the physical files on disk. Folder thumbnails are left untouched. File thumbnails are regenerated on demand the next time the corresponding media is viewed, so this only reclaims disk space and is safe to run.

**Music-video thumbnails are excluded.** They are the one kind generated only at scan time, so
deleting one would leave the Music view without a preview until the whole library was rescanned; use
the regeneration job below to rebuild any that an older cleanup already took.

**Request:** none

**Response:** `200 OK` — integer body with the number of file thumbnails deleted

```json
1580
```

**Use case:** Reclaim disk space, or force every photo/video thumbnail to be regenerated (for example after changing the thumbnail size setting).

---

### POST /api/thumbnails/music-videos/regenerate

Starts a background job that grabs a frame thumbnail for every music video that has none — the repair
path for the music-video thumbnails an older cleanup deleted, since unlike a photo or a regular video
a music video is never re-thumbnailed on demand. The job walks them in ascending id order with a
forward cursor (a file whose frame cannot be decoded still has none, so offset pages would hand it
back for ever) and returns as soon as it is started.

It refuses to start while a **scan** is running, since the scan makes these thumbnails itself.

**Response:** `202 Accepted` — the initial `MusicVideoThumbnailJobResponse`.

**Errors:** `409` when a scan is running or a regeneration is already in progress.

---

### POST /api/thumbnails/music-videos/regenerate/cancel

Requests cancellation of the running regeneration; it is honoured between batches. The thumbnails
already generated are kept — the job only ever visits music videos without one, so starting it again
simply carries on.

**Response:** `200 OK` — the state of the cancelling job.

---

### GET /api/thumbnails/music-videos/regenerate/status

Returns the state of the current or most recent regeneration: progress, elapsed time, an ETA, and the
counts generated/skipped/missing/failed. Any page may poll it at any time, including one opened long
after the job began.

**Response:** `200 OK` — `MusicVideoThumbnailJobResponse`, or an idle placeholder when none has run.

---

## Media Hashes

Base path: `/api/hashes`

### GET /api/hashes/stats

Returns the current content-hash coverage for image files.

**Request:** none

**Response:** `200 OK` — `HashStatsResponse`

```json
{
  "hashed": 950,
  "total": 1200
}
```

**Use case:** Display a hashing-progress indicator in the library folders view to show how many images have been processed by the scheduled hash-generation job.

---

### GET /api/hashes/stats/all

Returns the content-hash coverage for every media category at once, so the library view can render a
separate progress bar for each. Regular videos and music videos are reported separately even though
both are `VIDEO` files, since only their library folder's kind tells them apart.

**Response:** `200 OK` — `MediaHashStatsResponse`

```json
{
  "images":      { "hashed": 950,  "total": 1200 },
  "videos":      { "hashed": 40,   "total": 40 },
  "music":       { "hashed": 8100, "total": 8200 },
  "musicVideos": { "hashed": 300,  "total": 512 }
}
```

---

### POST /api/hashes/rehash

Starts a background job that recomputes the content hash of **every** media file and stores it when it
differs — the repair path for hashes made stale by a file being rewritten outside the application (or
by an edit made before edits refreshed the hash themselves).

The job walks every `MediaFile` in ascending id order with a forward cursor, since a rehashed row
still matches the query it was found by and offset pages would re-read it. A file that is missing or
unreadable **keeps** its stale hash rather than losing it, an out-of-date hash still being better than
none. It returns as soon as the job is started.

It refuses to start while a **scan**, a **tag import**, or a **deep clean** is running.

**Response:** `202 Accepted` — the initial `RehashJobResponse`.

**Errors:** `409` when one of those jobs is running, or a rehash is already in progress.

---

### POST /api/hashes/rehash/cancel

Requests cancellation of the running rehash; it is honoured between batches. The hashes already
corrected are kept — the job is idempotent, so it is simply started again.

**Response:** `200 OK` — the state of the cancelling job.

---

### GET /api/hashes/rehash/status

Returns the state of the current or most recent rehash: progress, elapsed time, an ETA, and the counts
processed/updated/unchanged/missing/failed. Any page may poll it at any time, including one opened
long after the rehash began.

**Response:** `200 OK` — `RehashJobResponse`, or an idle placeholder when none has run.

---

### GET /api/hashes/duplicates

Returns one page of media-file groups that are byte-for-byte duplicates of one another, detected by comparing the content hashes produced by the hash-generation job. Each group contains two or more files that share the same hash.

Results are paged over the duplicate **groups** (not the individual files), so a library with tens of thousands of duplicates is delivered in bounded chunks instead of a single large response. Groups are ordered by their shared hash, giving a stable order across page requests. Any file on the returned page that has no thumbnail yet has one generated on the fly so the duplicates view can show a preview of every copy.

**Query parameters**

| Parameter | Type    | Required | Description                                                                 |
|-----------|---------|----------|-----------------------------------------------------------------------------|
| `page`    | integer | No       | Zero-based page index. Default `0`; negative values are treated as `0`.     |
| `size`    | integer | No       | Number of duplicate groups per page. Default `50`, clamped to the range `1`–`200`. |

**Response:** `200 OK` — a Spring Data `Page` of `DuplicateGroupDTO`

```json
{
  "content": [
    {
      "hash": "a1b2c3…",
      "files": [
        {
          "id": 412,
          "fullPath": "/media/photos/vacation/img.jpg",
          "filename": "img.jpg",
          "thumbnailId": 88
        },
        {
          "id": 731,
          "fullPath": "/media/photos/temp/img.jpg",
          "filename": "img.jpg",
          "thumbnailId": 89
        }
      ]
    }
  ],
  "totalElements": 137,
  "totalPages": 3,
  "number": 0,
  "size": 50,
  "first": true,
  "last": false
}
```

`totalElements` is the total number of duplicate groups across all pages and `totalPages` is derived from it and the page `size`; a page index past the end returns an empty `content` array while still reporting the true totals.

**Use case:** Power the duplicates modal in the library folders view. The frontend requests 50 groups at a time, shows each group's copies with thumbnails and full paths, and lets the user delete redundant copies. Navigating to the next page requests the following 50 groups, so the browser never has to render the entire duplicate set at once.

---

## Uploads

Base path: `/api/uploads`

Writing into the library, so `ADMIN` only. A request larger than the configured multipart limits is
rejected by the servlet container with a **413** before either endpoint sees it.

### POST /api/uploads/music

Uploads one or more music files into the library and indexes them exactly as a scan would.

Each file is routed by its **own** media kind to the `WRITE` library folder whose `LibraryKind`
matches — an audio track to the writable `MUSIC` folder, a video to the writable `MUSIC_VIDEOS` one
(the oldest by id when several exist). Within it a file is filed under `year/month/day` taken from the
upload date (read once when the request starts, so a batch running across midnight is never split),
and an audio track additionally under an `artist - album (year)` directory read from its own tags, so
every track of one album lands in one directory and resolves to a single `Album` row. A track naming
no album stays in the day directory, as do music videos.

A stored file is then hashed (when `generateHashesOnScanning` is enabled), saved as a `MediaFile`,
tagged by the same scanner the scan would dispatch it to (creating the album, cover, or music-video
thumbnail), and its directory is recorded so the next scan skips it.

Files are accepted or rejected **individually**, so one rejection never holds back the rest: a
submitted name is reduced to its last path segment (it can never escape the destination), an existing
name is never overwritten (a numeric suffix is appended), and a file whose extension is not a
recognised audio/video format — or whose destination folder is not configured — is reported with an
explanatory message.

**Request:** `multipart/form-data` with one or more `files` parts.

**Response:** `200 OK` — `MediaUploadResultDTO`

```json
{
  "uploaded": 2,
  "rejected": 1,
  "items": [
    { "filename": "01 Song.mp3", "uploaded": true,  "destination": "2026/08/05/Alizée - Gourmandises (2000)/01 Song.mp3", "message": "Uploaded" },
    { "filename": "notes.txt",   "uploaded": false, "destination": null, "message": "Only audio tracks and music videos can be uploaded" }
  ]
}
```

**Errors:** `400` when no files were submitted.

---

### POST /api/uploads/screenshot

Stores a still frame captured from a playing music video (the player's **Z** shortcut) as an image in
the writable `PHOTOS` library folder, under a `year/month/day` directory taken from the capture date.

Unlike a music upload, the frame is **only written to disk** — no content hash, no `MediaFile` row, no
tag scan, no `Folder` record — because a screenshot is a file the user asked to keep, not library
content. (A later scan of a read folder containing that directory would of course index it like any
other image on disk.)

The requested name is reduced to a bare, safe name so it can never escape the destination, given an
image extension when it lacks one, and never overwrites an existing file — a `(1)` suffix is appended,
exactly as an upload does — so the name it was actually stored under is returned.

**Request:** `multipart/form-data`

| Part       | Type   | Required | Description                                             |
|------------|--------|----------|---------------------------------------------------------|
| `file`     | file   | yes      | The captured frame, an image of at most 25 MB           |
| `filename` | string | no       | Name to store the frame under; sanitized by the server  |

**Response:** `200 OK` — `ScreenshotUploadDTO`

```json
{
  "filename": "Alizée - Moi Lolita (1-23).jpg",
  "destination": "2026/08/05/Alizée - Moi Lolita (1-23).jpg"
}
```

**Errors:** `400` when the file is missing or empty, exceeds 25 MB, is not an image, no writable
photos library folder is configured, that folder does not exist on the server, or the file cannot be
written.

---

## Tag Export and Import

Base paths: `/api/exports` and `/api/imports`

An export carries the part of the library that lives **only** in the database — per-user ratings, view
counts, playlist memberships, the editable music tags, each item's date added, and an album's cover —
keyed by the file's **content hash**. The hash is what makes the file portable: a later import matches
by hash rather than by a path or a database id, neither of which survives a move to another
installation.

What a rescan can regenerate is deliberately left out (dimensions, capture date, camera, GPS, codec,
duration, file size, MIME type, thumbnails), since re-importing it could only overwrite fresher values
with stale ones. The file `path` is the one exception, exported so a third-party tool can locate the
file an entry describes; an import never writes it back.

### GET /api/exports/tags

Builds the export for one library kind. `PHOTOS` exports the images **and** the videos of the photos
libraries into one file (told apart by each item's `mediaKind`), `MUSIC` the audio tracks, and
`MUSIC_VIDEOS` the music videos.

An export carries every user's ratings and playlists, not only the caller's, so `/api/exports/**` is
restricted to `ADMIN` rather than following the blanket "any authenticated user may GET" rule.

**Query parameters**

| Parameter | Type   | Default | Description                                             |
|-----------|--------|---------|---------------------------------------------------------|
| `kind`    | string | —       | Library kind to export: `PHOTOS`, `MUSIC`, `MUSIC_VIDEOS` (case-insensitive, required) |

**Response:** `200 OK` — `MediaExportDTO`

```json
{
  "kind": "MUSIC",
  "exportedAt": "2026-08-05T18:20:11",
  "count": 8100,
  "skippedWithoutHash": 12,
  "skippedDuplicateHash": 4,
  "items": {
    "3f6a…": {
      "path": "2000/Gourmandises",
      "filename": "01 Moi Lolita.mp3",
      "songName": "Moi... Lolita",
      "artist": "Alizée",
      "genre": "Pop",
      "album": "Gourmandises",
      "year": 2000,
      "live": false,
      "source": "REGULAR",
      "coverPath": "2000/07/cover-1.jpg",
      "dateAdded": "2024-02-11T09:31:00",
      "users": {
        "yiannis": { "rating": 5, "views": 42, "lastViewDate": "2026-08-01T21:04:00", "playlists": ["Favourites"] }
      }
    }
  }
}
```

A record with no hash yet cannot be keyed and is skipped, as is one whose hash another item already
holds (two files sharing a hash are duplicate copies of the same content, so the first carries it);
both are counted in the envelope.

**Errors:** `400` when `kind` is missing or does not name a known library kind.

---

### POST /api/imports/tags

Uploads an export file whole and starts applying it in the background. An import of a large library
reads and writes for minutes, so tying it to a page staying open made it the browser's job to keep
going; started this way it is the application's, and the page only watches.

Entries are matched by content hash. What is restored: the music tag fields and each item's
`dateAdded`, per-user ratings, view counts, and last view date, playlists (created by name for the
named user when missing) and their memberships, and album covers from the export's covers directory —
an existing cover is kept when it is at least as good, so a re-run never downgrades artwork. Per-user
entries naming a user that does not exist here are counted and skipped.

Whether the file actually describes the chosen kind is settled by the job, which fails with an
explanatory message when it does not.

**Request:** `multipart/form-data` with a `file` part (the export file) and a `kind` parameter.

**Response:** `202 Accepted` — the initial `ImportJobResponse`.

**Errors:** `400` when `kind` is missing or unknown; `409` when an import is already running.

---

### POST /api/imports/tags/cancel

Requests cancellation of the running import. The chunks already written are kept, since running the
same file again writes only what still differs.

**Response:** `200 OK` — the state of the cancelling job.

---

### GET /api/imports/tags/status

Returns the state of the current or most recent import: phase, progress, elapsed time, an ETA, and the
counts of what it has restored so far. Any page may poll it at any time, including one opened long
after the import began.

**Response:** `200 OK` — `ImportJobResponse`, or an idle placeholder when none has run.

---

## Filesystem

Base path: `/api/filesystem`

Browses the server's filesystem so the frontend folder-browser can let the user pick a library folder path by navigating directories instead of typing it. Only directories are returned; regular files are never exposed, and hidden directories are omitted.

### GET /api/filesystem

Lists the immediate subdirectories of a directory on the server.

**Query parameters**

| Parameter | Type   | Required | Description                                                                       |
|-----------|--------|----------|-----------------------------------------------------------------------------------|
| `path`    | string | No       | Absolute directory path to list. When absent or blank the server user's home directory is listed as a starting point. |

**Response:** `200 OK` — `DirectoryListing`

```json
{
  "path": "/home/user",
  "parent": "/home",
  "directories": [
    { "name": "Documents", "path": "/home/user/Documents" },
    { "name": "Photos", "path": "/home/user/Photos" }
  ]
}
```

**Errors**

| Status | Condition                                      |
|--------|------------------------------------------------|
| 404    | Path does not exist or is not a directory      |

**Use case:** Power the folder-browser modal in the library folders view, letting the user navigate up to the parent directory and down into subdirectories before choosing a folder to add as a scan source.

---

## Logs

Read and maintenance access to the persistent application log. Every endpoint in this section
requires the `ADMIN` role; a non-admin receives **403**.

The log records meaningful backend events. Success and informational events (finished scan and
hash jobs, deep clean, login/registration, and user / library-folder / setting changes) are written
explicitly, while every `WARN` and `ERROR` raised by the application's own loggers is captured
automatically, so error cases appear here without being wired up individually.

### LogEntry schema

| Field         | Type    | Description                                                          |
|---------------|---------|---------------------------------------------------------------------|
| `id`          | long    | Primary key                                                         |
| `level`       | string  | `SUCCESS`, `INFO`, `WARN`, or `ERROR`                               |
| `category`    | string  | Coarse grouping, e.g. `SCAN`, `AUTH`, `USERS`; the originating class name for auto-captured events |
| `action`      | string  | Optional short action code, e.g. `SCAN_COMPLETED`; `null` for auto-captured events |
| `message`     | string  | Human-readable description                                          |
| `details`     | string  | Optional long-form detail such as an exception stack trace, or `null` |
| `username`    | string  | Username of the user who triggered the event, or `null` for system events |
| `dateCreated` | string  | ISO-8601 timestamp of when the event was recorded                   |

### GET /api/logs

Returns one page of log entries, newest first, narrowed by the optional filters.

**Query parameters**

| Parameter  | Type    | Default | Description                                                    |
|------------|---------|---------|----------------------------------------------------------------|
| `page`     | integer | `0`     | Zero-based page index                                          |
| `size`     | integer | `50`    | Entries per page (capped at 200)                              |
| `level`    | string  | —       | Restrict to a single level; blank or unknown means any        |
| `category` | string  | —       | Restrict to a single category; blank means any                |
| `search`   | string  | —       | Case-insensitive substring matched against the message        |

**Response** `200 OK` — a Spring `Page` of `LogEntry` (see [Common Schemas](#common-schemas) for the page envelope).

### GET /api/logs/categories

Returns the distinct, alphabetically sorted categories currently present in the log, for populating
the frontend filter.

**Response** `200 OK` — an array of strings.

### DELETE /api/logs

Deletes log entries. With no parameter the entire log is cleared; with `olderThanDays` only entries
older than that many days are removed.

**Query parameters**

| Parameter       | Type    | Description                                             |
|-----------------|---------|---------------------------------------------------------|
| `olderThanDays` | integer | Optional retention window; when present, only older entries are removed |

**Response** `204 No Content` when the whole log is cleared, or `200 OK` with `{ "deleted": <count> }` when a retention window was given.

**Use case:** Back the admin **Logs** page, which lists activity and errors with level/category/search filters, expandable stack traces, pagination, and a clear control.

---

## Radio

The **Radio** page's station catalogue, plus per-user favourites.

Stations come from the public [radio-browser.info](https://api.radio-browser.info) directory, but they
are **mirrored into the local database** rather than fetched per request: the directory is a
volunteer-run pool of mirrors that is regularly slow or unavailable, and querying it live meant a
timing-out mirror surfaced as a broken page. A background job crawls the whole directory (~62,000
stations) a batch at a time — 2,000 stations every 30 minutes by default, all three figures
configurable on the settings page — and resumes where it left off across restarts. Everything in this
section except the two endpoints noted below is answered from that local copy, so it keeps working
while the directory is down.

A stream itself is still not library content: it has no `MediaFile`, tag, hash, thumbnail, or play
count.

Two endpoints do leave the building per request, because neither can be answered from a database:
`POST /stations/{id}/click` (the usage report the directory asks for) and
`GET /stations/{id}/now-playing` (the title is broadcast inside the audio stream).

Reads require an authenticated user of any role. Favouriting and the click report are `POST`/`DELETE`
but are likewise allowed for any authenticated user — they are ordinary listening actions, scoped to
the caller. Triggering a catalogue sync is administration and requires `ADMIN`.

**The whole section is gated on the `radioEnabled` setting, which is `false` by default.** Radio is
the one feature that reaches a third-party service on its own — the catalogue crawl runs in the
background whether or not anybody opens the page — so it is opt-in. While it is off, the crawl never
runs and **every endpoint below answers `404 Not Found`**, exactly as an unknown station does, so a
disabled installation gives nothing away about the catalogue it is not keeping. The setting is read
per request, so switching it on from the settings page takes effect immediately, without a restart.

### RadioStation schema

| Field        | Type    | Description                                                          |
|--------------|---------|----------------------------------------------------------------------|
| `id`         | string  | The directory's stable station uuid                                  |
| `name`       | string  | Station name                                                         |
| `url`        | string  | Stream URL, already resolved past any playlist redirect              |
| `homepage`   | string  | Station website, or `null`                                           |
| `favicon`    | string  | Station logo URL, or `null`                                          |
| `tags`       | string  | Comma-separated genre tags as supplied upstream, or `null`           |
| `country`    | string  | Country name, or `null`                                              |
| `countryCode`| string  | ISO country code, or `null`                                          |
| `language`   | string  | Comma-separated broadcast languages, or `null`                       |
| `codec`      | string  | Stream codec such as `MP3` or `AAC`, or `null`                       |
| `bitrate`    | integer | Stream bitrate in kbps, or `0` when unknown                          |
| `votes`      | integer | The directory's community vote count                                 |
| `clickCount` | integer | How often the station was started through the directory recently     |
| `working`    | boolean | Whether the directory's last availability check succeeded            |
| `favorite`   | boolean | Whether the calling user has marked this station as a favourite      |

### RadioFilterOption schema

| Field          | Type    | Description                                    |
|----------------|---------|------------------------------------------------|
| `name`         | string  | The filter value as the directory spells it    |
| `stationCount` | integer | How many stations carry this value             |

### GET /api/radio/stations

Returns the stations matching the given filters, **the caller's favourites first**. With no filters
this is the catalogue's most-voted list with the user's favourites pulled to the top of it, which is
what the Radio page shows when it opens. Stations the last sync found the directory no longer
publishes are never returned (they are kept, deactivated, so favourites of them survive).

**Query parameters**

| Parameter       | Type    | Default | Description                                                       |
|-----------------|---------|---------|-------------------------------------------------------------------|
| `name`          | string  | —       | Case-insensitive substring matched against the station name       |
| `country`       | string  | —       | Exact country name; the value comes from `/countries`             |
| `tag`           | string  | —       | Exact genre tag; matched against whole tags, so `rock` does not match `rockabilly` |
| `language`      | string  | —       | Exact broadcast language; the value comes from `/languages`       |
| `favoritesOnly` | boolean | `false` | Restrict the listing to the caller's favourites                   |
| `order`         | string  | `votes` | Ordering field: `votes`, `clickcount`, `name`, or `bitrate`       |
| `reverse`       | boolean | `true`  | Reverse the ordering; for a count field this means highest first  |
| `offset`        | integer | `0`     | How many matching stations to skip, for paging                    |
| `limit`         | integer | `50`    | How many stations to return (capped at 200)                       |

Favourites lead the list whatever `order` is chosen, unless `favoritesOnly` is set (they are then all
favourites).

**Response** `200 OK` — an array of `RadioStation`.

### POST /api/radio/stations/{stationId}/favorite

Marks a station as one of the calling user's favourites, so it is listed first from now on. Allowed
for any authenticated user; favourites are per-user and one user never sees another's.

Idempotent — favouriting a station already favourited simply returns it as such.

**Response** `200 OK` — the `RadioStation`, with `favorite: true`. **404** when the catalogue holds no
such station.

### DELETE /api/radio/stations/{stationId}/favorite

Removes a station from the calling user's favourites. Idempotent, and likewise allowed for any
authenticated user.

**Response** `200 OK` — the `RadioStation`, with `favorite: false`. **404** when the catalogue holds no
such station.

### GET /api/radio/sync

Returns the state of the catalogue crawl: whether a batch is running, how far the current pass has
got (`fetchedThisCycle` against `directoryTotal`), when the last pass completed, and how many stations
the catalogue holds.

**RadioSyncStatus schema**

| Field              | Type    | Description                                                       |
|--------------------|---------|-------------------------------------------------------------------|
| `running`          | boolean | Whether a batch is being fetched at this moment                   |
| `cycleRunning`     | boolean | Whether a pass over the directory is in progress, between batches included |
| `lastCompletedAt`  | string  | ISO-8601 timestamp of the last completed pass, or `null`          |
| `stationCount`     | integer | How many stations the catalogue currently holds                   |
| `fetchedThisCycle` | integer | How many stations the current pass has fetched so far             |
| `directoryTotal`   | integer | How many stations the directory reports holding, or `0` if unknown |
| `added`            | integer | Stations added so far by the current pass                         |
| `updated`          | integer | Stations refreshed so far by the current pass                     |
| `deactivated`      | integer | Stations the last completed pass found the directory no longer publishes |
| `message`          | string  | Outcome of the last batch or pass, or the reason it failed        |

**Response** `200 OK` — a `RadioSyncStatus`.

### POST /api/radio/sync

Fetches the next batch of stations now, rather than waiting for the next scheduled one. Requires
`ADMIN`.

The catalogue is crawled a batch at a time on a timer; this brings the next batch forward, starting a
fresh pass when the last one has finished. Answers immediately with the status the batch started from,
since the work happens in the background, and a request made while a batch is already running is
refused rather than queued. A batch that fails leaves the offset where it was, so the next one retries
that window and the catalogue goes on serving what it holds.

**Response** `200 OK` — a `RadioSyncStatus`.

### GET /api/radio/countries

Returns the countries that have stations, most stations first, for the country filter. Counted from
the local catalogue by the daily sync.

**Query parameters**

| Parameter | Type    | Default | Description                        |
|-----------|---------|---------|------------------------------------|
| `limit`   | integer | `200`   | How many countries to return       |

**Response** `200 OK` — an array of `RadioFilterOption`.

### GET /api/radio/tags

Returns the most used genre tags, most stations first, for the genre filter. The upstream tag
vocabulary is user-supplied and enormous, so ordering by station count and taking the head is what
makes it usable as a dropdown. Counted from the local catalogue by the daily sync, which splits each
station's comma-separated tag list to do so.

**Query parameters**

| Parameter | Type    | Default | Description                   |
|-----------|---------|---------|-------------------------------|
| `limit`   | integer | `200`   | How many tags to return       |

**Response** `200 OK` — an array of `RadioFilterOption`.

### GET /api/radio/languages

Returns the broadcast languages that have stations, most stations first, for the language filter.

**Query parameters**

| Parameter | Type    | Default | Description                        |
|-----------|---------|---------|------------------------------------|
| `limit`   | integer | `200`   | How many languages to return       |

**Response** `200 OK` — an array of `RadioFilterOption`.

### GET /api/radio/stations/{stationId}/now-playing

Returns what the station is broadcasting right now, read live from the stream's own ICY
(Shoutcast/Icecast) metadata — the directory does not hold it, and a browser cannot read it, since it
needs an `Icy-MetaData: 1` request header and byte-level parsing of the audio stream.

The backend opens a short connection to the station, reads the first metadata block that carries a
title, and closes it; results are cached for 10 seconds so several listeners polling one station share
a single read. The station is resolved from its uuid through the directory, so only a stream the
directory itself publishes is ever connected to.

Always answers **200** with what is known, because a missing title is a normal state rather than an
error: `supported` is `false` when the station broadcasts no metadata at all (or the uuid is unknown),
and the title fields are `null` when it does but is currently sending none.

**RadioNowPlaying schema**

| Field       | Type    | Description                                                                 |
|-------------|---------|-----------------------------------------------------------------------------|
| `supported` | boolean | Whether the stream sends ICY metadata at all                                |
| `title`     | string  | The raw broadcast title, or `null`                                          |
| `artist`    | string  | The artist part of `title`, or `null` when it carries no recognisable one   |
| `song`      | string  | The song part of `title`, or the whole title when it could not be split     |

`artist` / `song` are the split of the conventional `Artist - Song` form on the first " - " (spaces
required, so `Blink-182` is not split). A title that does not follow it — many stations send a slogan
or their own name — is kept whole as `song` with a `null` artist rather than guessed at.

**Response** `200 OK` — a `RadioNowPlaying`.

### POST /api/radio/stations/{stationId}/click

Registers with the directory that the station was started, as its usage policy asks — those counts
are what its popularity ordering is built from. Allowed for any authenticated user.

Always answers `204`: the report is a courtesy to the directory and a failure is swallowed, so the
client never has to handle an error for a stream that is already playing.

**Response** `204 No Content`.

**Use case:** Back the **Radio** page, which browses the mirrored catalogue by name, country, genre,
language, and favourites. The picked stream is handed to the application's single persistent player — the same one the
library's music uses — so it keeps playing across navigation in the minimized bottom bar.
