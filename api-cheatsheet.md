# PhotoVault API — ściąga (PL)

Skrót do szybkiego sprawdzenia podczas pisania kodu. Pełna specyfikacja: [`api.md`](./api.md). Schema OpenAPI: [`api/openapi.yaml`](./api/openapi.yaml).

## Base URL i nagłówki

```
http://<pi-ip>:8080/v1/
Authorization: Bearer <accessToken>      ← wszędzie poza /auth/login i /health
Content-Type: application/json            ← request
                                             (poza POST /uploads → multipart/form-data)
Content-Type: application/json            ← response (success)
Content-Type: application/problem+json    ← response (error, RFC 7807)
```

## Konwencje

- ID jako opaque string z prefixem: `photo-`, `tag-`, `cat-`, `label-`, `user-`, `upload-`
- Daty: ISO-8601 UTC z `Z` na końcu (`"2026-04-18T17:24:30Z"`)
- JSON: `camelCase`
- URL: `kebab-case` dla wieloczłonowych segmentów
- Liczba mnoga w zasobach: `/photos`, `/tags`
- URL-e w response są względne (`/v1/photos/photo-abc/thumbnail`) — klient resolve do base

## Wszystkie endpointy (spis)

### Auth
```
POST   /v1/auth/login           → {accessToken, refreshToken, user}
POST   /v1/auth/refresh         → {accessToken, refreshToken}
POST   /v1/auth/logout          → 204
GET    /v1/auth/me              → UserDto
```

### Photos
```
GET    /v1/photos               → PhotoPage  (cursor, limit, q, tagIds, categoryIds,
                                              labelIds, favoritesOnly, uploadedBy=me|<id>)
GET    /v1/photos/{id}          → PhotoDto
PATCH  /v1/photos/{id}          → PhotoDto    body: {isFavorite?, tagIds?, categoryIds?, labelIds?}
DELETE /v1/photos/{id}          → 204

GET    /v1/photos/{id}/thumbnail → image/jpeg
GET    /v1/photos/{id}/medium    → image/jpeg
GET    /v1/photos/{id}/original  → image/*
```

### Tags
```
GET    /v1/tags                 → {items: [TagDto]}    (?usedOnly=true)
POST   /v1/tags                 → TagDto                body: {name}
PATCH  /v1/tags/{id}            → TagDto                body: {name}
DELETE /v1/tags/{id}            → 204
```

### Categories
```
GET    /v1/categories           → {items: [CategoryDto]}
POST   /v1/categories           → CategoryDto           body: {name, colorHex}
PATCH  /v1/categories/{id}      → CategoryDto           body: {name?, colorHex?}
DELETE /v1/categories/{id}      → 204
```

### Labels (read-only — system-defined)
```
GET    /v1/labels               → {items: [LabelDto]}
```

### Uploads
```
POST   /v1/uploads              → 202 + UploadDto       body: multipart {file, metadata?}
GET    /v1/uploads              → {items: [UploadDto]}  (?status=processing,uploading)
GET    /v1/uploads/{id}         → UploadDto
DELETE /v1/uploads/{id}         → 204                    (anuluj przed `done`)
```

### Users
```
GET    /v1/users                → {items: [UserDto]}
```

### Health
```
GET    /v1/health               → {status, version}     ← bez auth
```

## DTO — kształty

### PhotoDto
```json
{
  "id": "photo-abc123",
  "name": "zachod_morze.jpg",
  "sizeBytes": 4195000,
  "mimeType": "image/jpeg",
  "width": 4032, "height": 3024,
  "capturedAt": "2026-04-18T17:24:00Z",       // nullable
  "uploadedAt": "2026-04-18T17:24:30Z",
  "camera": "Pixel 8 Pro",                     // nullable
  "location": {                                 // nullable
    "latitude": 54.4641,
    "longitude": 18.5734,
    "placeName": "Sopot, PL"                    // nullable
  },
  "uploadedBy": {"id": "user-jarek", "displayName": "Jarek"},
  "tags": [TagDto],
  "categories": [CategoryDto],
  "labels": [LabelDto],
  "isFavorite": true,
  "processingStatus": "ready",                  // "processing" | "ready"
  "thumbnailUrl": "/v1/photos/photo-abc123/thumbnail",
  "mediumUrl": "/v1/photos/photo-abc123/medium",
  "originalUrl": "/v1/photos/photo-abc123/original"
}
```

### TagDto / CategoryDto / LabelDto
```json
{"id": "tag-001", "name": "#morze", "photoCount": 48}
{"id": "cat-001", "name": "Natura", "colorHex": "#FF8B45", "photoCount": 48}
{"id": "label-orange", "name": "orange", "colorHex": "#FF8B45", "photoCount": 23}
```

### UploadDto
```json
{
  "id": "upload-xyz789",
  "fileName": "IMG_xxx.jpg",
  "sizeBytes": 4195000,
  "uploadedBytes": 4195000,
  "status": "processing",          // created|uploading|processing|done|failed|cancelled
  "progress": 1.0,
  "photoId": null,                 // wypełniony gdy status=done
  "error": null,                   // wypełniony gdy status=failed
  "createdAt": "2026-04-18T17:24:30Z"
}
```

### UserDto / UserRefDto
```json
// pełny
{"id": "user-jarek", "username": "jarek", "displayName": "Jarek"}

// embedowany (np. w PhotoDto.uploadedBy) — bez username
{"id": "user-jarek", "displayName": "Jarek"}
```

### PhotoPage (paginacja)
```json
{
  "items": [PhotoDto, ...],
  "nextCursor": "eyJ1cGxv...",     // null gdy hasMore=false
  "hasMore": true
}
```

## Paginacja — cursor flow

```
1. GET /v1/photos                    → items + nextCursor=X
2. GET /v1/photos?cursor=X           → items + nextCursor=Y
3. GET /v1/photos?cursor=Y           → items + nextCursor=null + hasMore=false
```

Klient nie deszyfruje cursora. Zawiera (uploadedAt, id) ostatniego elementu strony. Stabilny pod współbieżnymi insertami.

## Filtry w `GET /v1/photos`

Wszystkie opcjonalne, można łączyć:
- `q=morze` — text search (nazwa, tagi, kategorie)
- `tagIds=tag-001,tag-002` — AND (musi mieć wszystkie)
- `categoryIds=cat-001,cat-002` — AND
- `labelIds=label-orange` — AND
- `favoritesOnly=true`
- `uploadedBy=user-jarek` lub `uploadedBy=me`
- `cursor=...&limit=30` — paginacja

## Upload flow (klient)

```
1. POST /v1/uploads multipart {file, metadata?: {tagIds, categoryIds, labelIds}}
   → 202 + UploadDto {status: "processing"}
2. (Pętla co 1-2 sek)
   GET /v1/uploads/{id} → UploadDto
3. Gdy status === "done":
   - Mamy UploadDto.photoId
   - Odśwież galerię (GET /v1/photos)
4. Gdy status === "failed":
   - Pokaż UploadDto.error
```

## Edycja tagów zdjęcia (PATCH semantyka set)

```
PATCH /v1/photos/photo-abc123
{"tagIds": ["tag-001", "tag-002"]}
```

Lista **zastępuje** dotychczasowe tagi (set semantics, nie append). Żeby dodać jeden tag:
1. Czytasz aktualne `tags` ze zdjęcia
2. Wyciągasz `id` każdego, dodajesz nowy
3. Wysyłasz pełną listę

To samo dla `categoryIds` i `labelIds`. Wszystkie pola w PATCH są opcjonalne.

## Statusy HTTP

| Kod | Kiedy |
|----:|-------|
| 200 | OK z body |
| 201 | Created (POST tworzący zasób; zwraca też `Location` header) |
| 202 | Accepted (POST /uploads — bytes mam, processing w tle) |
| 204 | OK bez body (DELETE, logout) |
| 400 | Validation failed |
| 401 | Brak / zły / wygasły token |
| 403 | Token OK ale brak uprawnień (na razie nie używane) |
| 404 | Nie znaleziono |
| 409 | Conflict (duplicate name, invalid state transition) |
| 413 | Plik za duży |
| 415 | Zły mime type |
| 423 | Locked — np. thumbnail jeszcze się generuje |
| 500 | Błąd serwera |
| 503 | Maintenance / restart |

## Errors — RFC 7807 Problem Details

```json
{
  "type": "https://photovault.local/errors/photo-not-found",
  "title": "Photo Not Found",
  "status": 404,
  "detail": "Photo with id 'photo-abc123' does not exist",
  "instance": "/v1/photos/photo-abc123"
}
```

Walidacja dodaje `errors`:
```json
{
  "type": "https://photovault.local/errors/validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Request body contains invalid fields",
  "instance": "/v1/tags",
  "errors": {
    "name": ["must not be blank", "must start with #"]
  }
}
```

### Slugi błędów (do mapowania w kliencie)

```
not-found, photo-not-found, tag-not-found, category-not-found,
label-not-found, upload-not-found, user-not-found
validation-failed, invalid-cursor
unauthenticated, invalid-token, invalid-credentials
forbidden
duplicate-tag-name, duplicate-category-name
invalid-state-transition
upload-too-large, unsupported-file-type
processing-not-ready
internal-error, service-unavailable
```

Klient parsuje slug z końca `type` URI: `.../errors/<slug>`. Mapuje na `DomainError` w `:core:domain`.

## Auth flow (klient)

```
1. App start:
   - Czytaj token z secure storage (EncryptedSharedPreferences)
   - GET /v1/auth/me z token
     - 200 → zalogowany; przejdź do galerii
     - 401 → próbuj refresh
2. Refresh:
   POST /v1/auth/refresh {refreshToken}
     - 200 → zapisz nowe tokeny, retry oryginalny request
     - 401 → wyczyść storage, pokaż login screen
3. Login:
   POST /v1/auth/login {username, password}
     - 200 → zapisz tokeny w secure storage
     - 401 → pokaż "invalid credentials"
4. Logout:
   POST /v1/auth/logout
   Wyczyść tokeny lokalnie (zawsze, niezależnie od response)
```

Każdy zwykły request opakowany w interceptor:
- Wstaw `Authorization: Bearer <accessToken>`
- Przy 401 z `type: invalid-token` → próbuj refresh + retry raz
- Przy 401 z `type: invalid-credentials` (tylko z /auth/login) → propaguj do UI

## Wersjonowanie

Forward-compatible w `/v1/`:
- ✅ Nowe pole w response — klient ignoruje nieznane
- ✅ Nowy endpoint
- ✅ Nowy opcjonalny query param
- ✅ Nowy slug errora

Breaking → `/v2/`:
- ❌ Zmiana typu pola
- ❌ Usunięcie / rename pola
- ❌ Zmiana semantyki istniejącego pola
- ❌ Zmiana defaultów
