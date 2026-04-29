# PhotoVault REST API

Version: `v1`
Status: **draft** — phase 2 of the project roadmap (contract design).

This document describes the REST API contract between the PhotoVault Android client and the self-hosted Ktor backend. It is the single source of narrative truth for the contract; the machine-readable counterpart lives in [`api/openapi.yaml`](./api/openapi.yaml).

The contract is implemented in three places:

1. **Client** — `:core:network` module of the Android app (Ktor client + DTOs).
2. **Server** — Kotlin + Ktor running in Docker on a Raspberry Pi.
3. **Mock** — n8n webhook flow during phase 2, before the Ktor server exists.

All three must agree on the shapes defined here. When in doubt, this document wins; when this document is wrong, file an issue and update it before changing code.

---

## Table of contents

1. [Conventions](#conventions)
2. [Authentication](#authentication)
3. [Resources](#resources)
   - [Photos](#photos)
   - [Tags](#tags)
   - [Categories](#categories)
   - [Labels](#labels)
   - [Uploads](#uploads)
   - [Users](#users)
4. [Pagination](#pagination)
5. [Errors](#errors)
6. [Versioning](#versioning)
7. [Health check](#health-check)
8. [Design rationale](#design-rationale)

---

## Conventions

### Base URL

All endpoints are prefixed with `/v1/`. The full base URL depends on deployment — typically `http://<pi-ip>:8080/v1/` on the LAN, or a Tailscale-resolved hostname for remote access. The client is configured with the base URL once; relative URLs in JSON responses are resolved against it.

### Content types

- Request bodies: `application/json` unless explicitly noted (uploads use `multipart/form-data`).
- Response bodies: `application/json` for resources, `application/problem+json` for errors (RFC 7807), `image/*` for binary photo content.
- Character encoding: UTF-8 throughout.

### Dates and times

All timestamps are ISO-8601 strings in UTC, with the `Z` suffix:

```
"uploadedAt": "2026-04-18T17:24:30Z"
```

The client converts to local time for display. The server stores in UTC.

### Identifiers

Resource IDs are opaque strings, prefixed with the resource type:

- Photos: `photo-<random>`
- Tags: `tag-<random>`
- Categories: `cat-<random>`
- Labels: `label-<name>` (deterministic for system-defined labels: `label-red`, `label-orange`, etc.)
- Users: `user-<username>`
- Uploads: `upload-<random>`

The client treats IDs as opaque — never parses or constructs them, only stores and forwards.

### URLs in responses

When a resource references binary content (photo files), the server returns **relative** URLs:

```json
{
  "thumbnailUrl": "/v1/photos/photo-abc123/thumbnail"
}
```

The client resolves these against its configured base URL. This decouples response data from the deployment hostname.

### Naming

- JSON field names use `camelCase` (matches Kotlin conventions; `kotlinx.serialization` doesn't need adapters).
- URL paths use `kebab-case` for multi-word segments (none in this API yet, but the convention applies).
- Query parameters use `camelCase`.

---

## Authentication

The API uses JWT bearer tokens. All endpoints except `POST /v1/auth/login` and `GET /v1/health` require authentication.

### Login

```http
POST /v1/auth/login
Content-Type: application/json

{
  "username": "jarek",
  "password": "redacted"
}
```

**Success — 200 OK:**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user-jarek",
    "username": "jarek",
    "displayName": "Jarek"
  }
}
```

**Failure — 401 Unauthorized** (invalid credentials, see [Errors](#errors)).

### Token usage

Authenticated requests include:

```http
Authorization: Bearer <accessToken>
```

The access token is short-lived (1 hour by default). Once it expires, the server returns 401 with `type: invalid-token`. The client uses the refresh token to obtain a new pair.

### Refresh

```http
POST /v1/auth/refresh
Content-Type: application/json

{
  "refreshToken": "eyJhbGc..."
}
```

**Success — 200 OK** with a new `{accessToken, refreshToken}` pair. The old refresh token is invalidated; clients must store the new one.

**Failure — 401** if the refresh token is invalid, expired, or has been revoked (e.g., by logout). Client must re-authenticate via `POST /v1/auth/login`.

### Logout

```http
POST /v1/auth/logout
Authorization: Bearer <accessToken>
```

**Success — 204 No Content.** The server invalidates the refresh token paired with this session. The client clears tokens from secure storage.

### Current user

```http
GET /v1/auth/me
Authorization: Bearer <accessToken>
```

**Success — 200 OK** with `UserDto`. Used by the client on app startup to verify the token is still valid and to populate the current-user state.

### User accounts

User accounts are created administratively (direct database insert or a server CLI command), not through the API. There is **no self-registration endpoint**. This is a single-household self-hosted tool, not a public service.

---

## Resources

### Photos

The central resource. A photo is a binary file (JPEG/PNG/HEIC) plus metadata, uploaded by a specific user, optionally annotated with tags, categories, and labels.

#### `GET /v1/photos`

List photos, paginated.

**Query parameters:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `cursor` | string | — | Opaque cursor from a previous response. Omitted on first page. |
| `limit` | int | 30 | Page size. Max: 100. |
| `q` | string | — | Free-text search across name, tag names, category names. |
| `tagIds` | string (csv) | — | Filter to photos that have **all** listed tags (AND). |
| `categoryIds` | string (csv) | — | Filter to photos in **all** listed categories (AND). |
| `labelIds` | string (csv) | — | Filter by labels (AND). |
| `favoritesOnly` | bool | false | If true, return only favorited photos. |
| `uploadedBy` | string | — | Filter by uploader user ID. Special value: `me` resolves to the authenticated user. |

**Success — 200 OK:**

```json
{
  "items": [
    {
      "id": "photo-abc123",
      "name": "zachod_morze.jpg",
      "sizeBytes": 4195000,
      "mimeType": "image/jpeg",
      "width": 4032,
      "height": 3024,
      "capturedAt": "2026-04-18T17:24:00Z",
      "uploadedAt": "2026-04-18T17:24:30Z",
      "camera": "Pixel 8 Pro",
      "location": {
        "latitude": 54.4641,
        "longitude": 18.5734,
        "placeName": "Sopot, PL"
      },
      "uploadedBy": {
        "id": "user-jarek",
        "displayName": "Jarek"
      },
      "tags": [
        {"id": "tag-001", "name": "#zachód-słońca", "photoCount": 12},
        {"id": "tag-002", "name": "#morze", "photoCount": 48}
      ],
      "categories": [
        {"id": "cat-001", "name": "Natura", "colorHex": "#FF8B45", "photoCount": 48}
      ],
      "labels": [
        {"id": "label-orange", "name": "orange", "colorHex": "#FF8B45", "photoCount": 23}
      ],
      "isFavorite": true,
      "processingStatus": "ready",
      "thumbnailUrl": "/v1/photos/photo-abc123/thumbnail",
      "mediumUrl": "/v1/photos/photo-abc123/medium",
      "originalUrl": "/v1/photos/photo-abc123/original"
    }
  ],
  "nextCursor": "eyJ1cGxvYWRlZEF0IjoiMjAyNi0wNC0xOFQxNzoyNDowMFoiLCJpZCI6InBob3RvLWFiYzEyMyJ9",
  "hasMore": true
}
```

If there are no more pages: `nextCursor: null`, `hasMore: false`.

Photos are sorted by `uploadedAt` descending (newest first), then by `id` for stable ordering when timestamps collide.

#### `GET /v1/photos/{id}`

Fetch a single photo by ID.

**Success — 200 OK** with a single `PhotoDto` (same shape as items in the list).

**Errors:** 404 if the photo doesn't exist.

#### `PATCH /v1/photos/{id}`

Partially update a photo's metadata. All fields are optional; omitted fields are unchanged.

**Request:**

```json
{
  "isFavorite": true,
  "tagIds": ["tag-001", "tag-002"],
  "categoryIds": ["cat-001"],
  "labelIds": ["label-orange"]
}
```

When `tagIds` is provided, it **replaces** the current tag list (set semantics). Same for `categoryIds` and `labelIds`. To add a single tag, the client reads current tags, appends, and sends the full list.

**Success — 200 OK** with the updated `PhotoDto`.

**Errors:**

- 404 if the photo doesn't exist
- 400 (`type: validation-failed`) if any referenced tag/category/label ID doesn't exist

#### `DELETE /v1/photos/{id}`

Delete a photo. Removes the file, thumbnail, medium, and all metadata. Cascades to junction tables (photo↔tag, etc.).

**Success — 204 No Content.**

**Errors:** 404 if the photo doesn't exist.

#### `GET /v1/photos/{id}/thumbnail`

Fetch the small thumbnail (typically 200×200, server-decided).

**Success — 200 OK** with `Content-Type: image/jpeg` and the binary content.

**Errors:**

- 404 if the photo doesn't exist
- 423 Locked (`type: processing-not-ready`) if the photo's `processingStatus` is still `processing`

#### `GET /v1/photos/{id}/medium`

Fetch the medium-sized image, suitable for the detail screen.

Same shape as thumbnail. Same error model.

#### `GET /v1/photos/{id}/original`

Fetch the original, unmodified upload.

Same shape. Always available regardless of `processingStatus` (the original is the first thing the server stores).

---

### Tags

User-defined text labels with a `#` prefix (e.g., `#morze`, `#zachód-słońca`). Tags are first-class resources — they exist independently of any photo.

#### `GET /v1/tags`

List all tags.

**Query parameters:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `usedOnly` | bool | false | If true, return only tags that are assigned to at least one photo. |

**Success — 200 OK:**

```json
{
  "items": [
    {"id": "tag-001", "name": "#morze", "photoCount": 48},
    {"id": "tag-002", "name": "#zachód-słońca", "photoCount": 12}
  ]
}
```

No pagination — there will rarely be more than a few hundred tags.

#### `POST /v1/tags`

Create a new tag.

**Request:**

```json
{"name": "#sunset"}
```

`name` must start with `#`, must not be blank, and must be unique (case-insensitive).

**Success — 201 Created** with the new `TagDto` and `Location` header pointing to `/v1/tags/{id}`.

**Errors:**

- 400 (`type: validation-failed`) — name is blank, missing `#`, or invalid characters
- 409 Conflict (`type: duplicate-tag-name`) — a tag with this name already exists

#### `PATCH /v1/tags/{id}`

Rename a tag.

**Request:**

```json
{"name": "#newname"}
```

**Success — 200 OK** with the updated `TagDto`. The tag remains attached to all the same photos.

**Errors:** 404, 400, 409 (same conditions as POST).

#### `DELETE /v1/tags/{id}`

Delete a tag. Removes from all photos that have it.

**Success — 204 No Content.**

**Errors:** 404.

---

### Categories

User-defined categories with associated colors. Same model as tags, plus a `colorHex` field.

```
GET    /v1/categories                         → list all
POST   /v1/categories                         → create new
PATCH  /v1/categories/{id}                    → update name and/or color
DELETE /v1/categories/{id}                    → delete
```

**`CategoryDto`:**

```json
{
  "id": "cat-001",
  "name": "Natura",
  "colorHex": "#FF8B45",
  "photoCount": 48
}
```

`colorHex` must be a valid 7-character hex string (`#RRGGBB`).

Validation, errors, and semantics mirror tags exactly, with `duplicate-category-name` instead of `duplicate-tag-name`.

---

### Labels

System-defined color labels. Unlike tags and categories, the user does not create or delete them — there is a fixed set of six (red, orange, yellow, green, blue, purple).

#### `GET /v1/labels`

```json
{
  "items": [
    {"id": "label-red", "name": "red", "colorHex": "#E53935", "photoCount": 5},
    {"id": "label-orange", "name": "orange", "colorHex": "#FF8B45", "photoCount": 23},
    {"id": "label-yellow", "name": "yellow", "colorHex": "#FDD835", "photoCount": 8},
    {"id": "label-green", "name": "green", "colorHex": "#43A047", "photoCount": 14},
    {"id": "label-blue", "name": "blue", "colorHex": "#1E88E5", "photoCount": 11},
    {"id": "label-purple", "name": "purple", "colorHex": "#8E24AA", "photoCount": 3}
  ]
}
```

There is no POST/PATCH/DELETE on `/v1/labels`. Attempting one returns 405 Method Not Allowed.

---

### Uploads

Uploads are first-class resources representing the lifecycle of a file being added to the library: from initial transfer through server-side processing (thumbnail generation, metadata extraction) to availability as a finished photo.

#### Upload lifecycle

```
created → uploading → processing → done
                  ↘   ↘
                    failed / cancelled
```

- **created** — record exists, transfer not yet started (rare; usually skipped)
- **uploading** — bytes are being received
- **processing** — bytes received in full; server is generating thumbnails, extracting EXIF, etc.
- **done** — finished; `photoId` is populated and the photo is visible in `GET /v1/photos`
- **failed** — error during upload or processing; `error` field populated
- **cancelled** — client called DELETE while in progress

#### `POST /v1/uploads`

Start a new upload. The body is `multipart/form-data` with two parts:

- `file` (required) — the binary file (JPEG, PNG, or HEIC)
- `metadata` (optional) — JSON object with `tagIds`, `categoryIds`, `labelIds` to apply to the resulting photo

**Request example:**

```
POST /v1/uploads HTTP/1.1
Authorization: Bearer ...
Content-Type: multipart/form-data; boundary=----photovault
Content-Length: ...

------photovault
Content-Disposition: form-data; name="file"; filename="IMG_20260418_172400.jpg"
Content-Type: image/jpeg

<binary data>
------photovault
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

{"tagIds": ["tag-002"], "categoryIds": ["cat-001"]}
------photovault--
```

**Success — 202 Accepted:**

```json
{
  "id": "upload-xyz789",
  "fileName": "IMG_20260418_172400.jpg",
  "sizeBytes": 4195000,
  "uploadedBytes": 4195000,
  "status": "processing",
  "progress": 1.0,
  "photoId": null,
  "error": null,
  "createdAt": "2026-04-18T17:24:30Z"
}
```

The server returns 202 once it has the bytes and has started background processing. The client polls `GET /v1/uploads/{id}` until `status` is `done` or `failed`.

**Errors:**

- 400 (`type: validation-failed`) — missing file, invalid metadata JSON
- 413 Payload Too Large (`type: upload-too-large`) — file exceeds server limit (default: 50 MB)
- 415 Unsupported Media Type (`type: unsupported-file-type`) — not JPEG/PNG/HEIC

#### `GET /v1/uploads`

List active and recently completed uploads, optionally filtered by status.

**Query parameters:**

| Name | Type | Description |
|------|------|-------------|
| `status` | string (csv) | Filter to specific statuses, e.g., `?status=processing,uploading` |

**Success — 200 OK** with `{items: [UploadDto, ...]}`. No pagination; the list is naturally bounded.

#### `GET /v1/uploads/{id}`

Fetch a single upload's status. Used for polling.

**Success — 200 OK** with `UploadDto`.

**Errors:** 404.

#### `DELETE /v1/uploads/{id}`

Cancel an upload. Allowed only when `status` is `created`, `uploading`, or `processing`. After the upload reaches `done`, use `DELETE /v1/photos/{photoId}` instead.

**Success — 204 No Content.**

**Errors:**

- 404 — upload doesn't exist
- 409 Conflict (`type: invalid-state-transition`) — upload is already `done`, `failed`, or `cancelled`

---

### Users

Read-only resource for now. User accounts are managed administratively (direct DB or server CLI), not via API.

#### `GET /v1/users`

List all users in the system.

**Success — 200 OK:**

```json
{
  "items": [
    {"id": "user-jarek", "username": "jarek", "displayName": "Jarek"},
    {"id": "user-partnerka", "username": "partnerka", "displayName": "Partnerka"}
  ]
}
```

Used by the client to populate the "uploaded by" filter dropdown.

---

## Pagination

`GET /v1/photos` is the only paginated endpoint at the moment. Tags, categories, labels, and users return full lists.

### Cursor format

The cursor is an opaque base64-encoded string. The client treats it as opaque — never decodes, never constructs.

Internally, the cursor encodes `{uploadedAt, id}` of the last item on the page, allowing the next query to be:

```sql
WHERE (uploadedAt, id) < (?, ?) ORDER BY uploadedAt DESC, id DESC LIMIT ?
```

This makes pagination **stable under concurrent inserts** — adding a photo while paging won't shift positions or duplicate results.

### Why no `totalCount`?

Counting all matching rows on every paginated request is expensive (a separate `COUNT(*)` query). Infinite scroll doesn't need a total — the user scrolls until `hasMore` is false. If a counter is needed for UI (e.g., "1247 photos"), we'll add a dedicated endpoint like `GET /v1/photos/count?...` that the client calls once.

---

## Errors

All errors follow [RFC 7807 Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807). The response Content-Type is `application/problem+json`.

### Standard fields

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | URI identifying the error type (also returned as a short slug; see below). |
| `title` | string | Short, human-readable summary of the error. |
| `status` | int | HTTP status code (mirrors the response status). |
| `detail` | string | Human-readable explanation specific to this occurrence. |
| `instance` | string | URI of the request that caused the error. |

### Validation extension

For 400 errors caused by request validation, the response includes an `errors` extension member:

```json
{
  "type": "https://photovault.local/errors/validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Request body contains invalid fields",
  "instance": "/v1/tags",
  "errors": {
    "name": ["must not be blank", "must start with #"],
    "colorHex": ["must be a valid hex color"]
  }
}
```

`errors` is a map from field name to a list of validation messages.

### Error catalog

| Type slug | HTTP | When |
|-----------|------|------|
| `not-found` | 404 | Generic "resource not found" |
| `photo-not-found` | 404 | Photo specifically |
| `tag-not-found` | 404 | Tag specifically |
| `category-not-found` | 404 | Category specifically |
| `label-not-found` | 404 | Label specifically |
| `upload-not-found` | 404 | Upload specifically |
| `user-not-found` | 404 | User specifically |
| `validation-failed` | 400 | Request body or query parameter validation failed |
| `invalid-cursor` | 400 | Pagination cursor is malformed or stale |
| `unauthenticated` | 401 | Missing or invalid `Authorization` header |
| `invalid-token` | 401 | Token is expired, malformed, or revoked |
| `invalid-credentials` | 401 | Login failed |
| `forbidden` | 403 | Authenticated but not allowed (reserved for future role-based access) |
| `duplicate-tag-name` | 409 | Tag name already exists |
| `duplicate-category-name` | 409 | Category name already exists |
| `invalid-state-transition` | 409 | Operation not allowed in current resource state (e.g., cancel finished upload) |
| `upload-too-large` | 413 | File exceeds server's max upload size |
| `unsupported-file-type` | 415 | File MIME type is not allowed |
| `processing-not-ready` | 423 | Photo's derived assets (thumbnail, medium) are still being generated |
| `internal-error` | 500 | Server-side error |
| `service-unavailable` | 503 | Server is starting up, restarting, or overloaded |

The full URI for each is `https://photovault.local/errors/<slug>`. The base hostname (`photovault.local`) is symbolic — the client matches on the slug at the end of the URI.

---

## Versioning

The API is versioned by URL prefix: `/v1/`. Future incompatible changes will introduce `/v2/`, `/v3/`, etc. Old versions remain available during a deprecation window.

### Forward-compatible changes (no version bump)

The following changes are made within `/v1/` without breaking existing clients:

- Adding new endpoints
- Adding new fields to response bodies (clients ignore unknown fields by default with `kotlinx.serialization`)
- Adding new optional query parameters
- Adding new optional fields to request bodies
- Adding new error type slugs (clients fall back to status code for unknown slugs)
- Loosening validation (accepting more inputs)

### Breaking changes (require new version)

These would require `/v2/`:

- Removing or renaming fields
- Changing field types (e.g., `string` → `int`)
- Changing semantic meaning of an existing field
- Removing endpoints
- Tightening validation (rejecting previously valid inputs)
- Changing default values

---

## Health check

#### `GET /v1/health`

Public endpoint (no auth required) for connectivity verification.

**Success — 200 OK:**

```json
{
  "status": "ok",
  "version": "1.0.0"
}
```

`version` is the server build version, useful for debugging client/server mismatches. The client may call this on app startup to validate that the configured base URL points to a running PhotoVault server.

---

## Design rationale

A few decisions worth documenting because they may look surprising at first glance:

### Why denormalize tags/categories/labels into `Photo`?

A photo response includes full `Tag` / `Category` / `Label` objects, not just IDs. This is denormalization — the same tag appears in many photo responses. We accept this because:

- The client almost always needs the names and colors when displaying a photo. Returning IDs only would require a separate lookup endpoint or pre-fetched cache.
- Tags/categories rarely change. Stale denormalized data isn't a real problem.
- The data set is small (single-household scale), so payload size is negligible.

The local Room database normalizes via junction tables, and `:core:data` performs the denormalization on the way out.

### Why are uploads asynchronous?

The server generates thumbnails and medium-sized images during the upload flow. These operations take time (image decoding, resizing, re-encoding). If the upload were synchronous, the client would wait on a long request, which is poor UX and risks timeouts.

By making upload async with `202 Accepted` and a polling endpoint, the client gets immediate feedback ("upload received, processing"), and the server can do work at its own pace. The cost is one extra round trip, paid asynchronously.

### Why `PATCH` with full lists for tags?

`PATCH /v1/photos/{id}` with `tagIds: [...]` replaces the tag list (set semantics). The alternative — `POST /v1/photos/{id}/tags/{tagId}` per addition and `DELETE` per removal — was rejected because:

- The natural client UX is "edit tags" (open a multi-select, save). The endpoint shape mirrors the UX.
- Per-tag operations would multiply round trips for typical edits.
- Set semantics are simple and idempotent.

### Why no self-registration?

PhotoVault is a self-hosted single-household tool. Public registration would be an attack surface for no benefit. Accounts are created administratively when the household sets up the server. If multi-tenant deployment becomes a goal, registration would be added — but as an explicitly enabled feature, not a default.

### Why JWT instead of session cookies?

The client is a native Android app, not a browser. Cookies are awkward for native apps — they require a cookie jar, are tied to URLs in non-obvious ways, and complicate testing. JWTs are explicit headers, easy to log and reproduce in `curl` or Postman, and library-supported in both Ktor server (auth-jwt plugin) and Ktor client (Auth plugin).

### Why hide pagination behind `cursor` instead of `offset`?

Two reasons: stability under concurrent inserts (cursor-based pagination doesn't shift when new photos arrive during scrolling), and performance for large datasets (cursor lookups are O(log n) via the index, while large offsets are O(n) — SQLite has to scan past `OFFSET` rows). Infinite scroll doesn't need "go to page 5", so offset's main feature is unused.

---

## Change log

This document is versioned alongside the code. Significant changes are recorded here.

| Date | Change |
|------|--------|
| 2026-04-29 | Initial draft, v1 of the API contract |
