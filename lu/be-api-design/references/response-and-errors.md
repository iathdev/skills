# Response Envelope & Error Handling

## Response Envelope

All endpoints use the unified envelope from `internal/shared/response/`:

```json
{
  "success": true,
  "data": {},
  "meta": {
    "request_id": "01968ed7-...",
    "timestamp": "2026-04-04T10:30:00Z"
  }
}
```

`data` and `error` are **mutually exclusive** — omitted (not null) on the other case.

### Success

```go
response.Success(c, http.StatusOK, data)
response.Success(c, http.StatusCreated, data)
response.Success(c, http.StatusOK, nil)  // DELETE

response.SuccessList(c, items, response.OffsetPaginationMeta{
    Total: total, Page: page, PageSize: pageSize, TotalPages: totalPages,
})

response.SuccessCursorList(c, items, response.CursorPaginationMeta{
    NextCursor: cursor, HasNext: true, Limit: 20,
})
```

### Errors

```go
// From use cases (via AppError)
response.HandleError(c, err)

// Direct (middleware / router, no AppError)
response.BadRequest(c, "common.bad_request")
response.Unauthorized(c, "common.unauthorized")
response.NotFound(c, "common.route_not_found")
response.ValidationError(c, bindingErr)  // 422 with field-level details
```

## Error Object

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Không tìm thấy từ vựng",
    "details": {}
  },
  "meta": { "request_id": "...", "timestamp": "..." }
}
```

- `code` — SCREAMING_SNAKE_CASE (from `apperr.Code`)
- `message` — i18n-translated at response layer
- `details` — omitted for simple errors

## AppError Constructors

**4xx — no cause:**

| Constructor | Status | When |
|---|---|---|
| `apperr.BadRequest(key)` | 400 | Invalid input, FK violation |
| `apperr.Unauthorized(key)` | 401 | Auth failed |
| `apperr.Forbidden(key)` | 403 | Not allowed |
| `apperr.NotFound(key)` | 404 | Resource doesn't exist |
| `apperr.Conflict(key)` | 409 | Duplicate |
| `apperr.ValidationFailed(key)` | 422 | Domain validation |
| `apperr.TooManyRequests(key)` | 429 | Rate limit |

**5xx — always carry cause:**

| Constructor | Status | When |
|---|---|---|
| `apperr.InternalServerError(key, err)` | 500 | Unexpected system errors |
| `apperr.ServiceUnavailable(key, err)` | 503 | External service down |

### Granular Business Codes

Use `WithCode()` to override the default code when the client needs to distinguish
a specific business failure from the generic HTTP-class code:

```go
apperr.Forbidden("entitlement.not_entitled").
    WithCode("FEATURE_NOT_ENTITLED").
    WithData(map[string]any{"feature": "ocr_scan", "current_plan": "free"})

apperr.BadRequest("common.invalid_cursor").WithCode("INVALID_CURSOR")
```

Common overrides in lu-backend:
- `INVALID_CURSOR` (400) — cursor malformed / version mismatch / sort mismatch

## Error Flow

```
Repository (raw error / nil) → Use Case (wraps into AppError) → Handler (response.HandleError) → HTTP status + i18n
```

### Repository Error Convention

- **Not found** → `(nil, nil)`. Use case: `apperr.NotFound(key)`
- **Other errors** → raw `error`. Use case: `apperr.InternalServerError(key, err)`

### i18n Keys, Not English

```go
// BAD
apperr.InternalServerError("failed to save", err)
// GOOD
apperr.NotFound("vocabulary.not_found")
```

Keys: `<subdomain>.<key_name>`. Files: `internal/shared/i18n/translations/<lang>/<subdomain>.json`.

## Meta Object

Always present. `pagination` only for list endpoints.

| Field | Type | Presence |
|---|---|---|
| `request_id` | UUID v7 | always |
| `timestamp` | RFC 3339 | always |
| `pagination` | object | list only |

## Pagination

### Offset (`type: "offset"`)

```
GET /vocabulary?page=2&page_size=20
```

| Field | Type | Description |
|---|---|---|
| `total` | int | Total items |
| `page` | int | Current page (1-based) |
| `page_size` | int | Items per page |
| `total_pages` | int | Total pages |

Defaults: `page=1`, `page_size=10`. Constraints: `min=1`, `max=100`.

### Cursor (`type: "cursor"`)

```
GET /collections?cursor=eyJ2IjoxLCJzIjoi...&limit=20
```

| Field | Type | Description |
|---|---|---|
| `next_cursor` | string | Opaque base64url cursor (omitted on last page) |
| `has_next` | bool | Whether more items exist |
| `limit` | int | Items per page |

**Internal format** (clients must NOT decode — round-trip verbatim):

Cursor is `base64url(JSON)` of a typed struct with:
- `v` (int) — version; bump when changing cursor shape so in-flight pagers fail fast
- `s` (string) — the `sort=` key the cursor was minted under
- `t` / `n` / `id` — the keyset anchor (last row's sort column + ID tiebreaker)

Decode rejects the cursor (400 `INVALID_CURSOR`) if:
- malformed base64 or JSON
- `v` != current version
- `s` != the current `sort=` query param (client changed sort mid-pagination)
- missing anchor field for the requested sort

Use the `util.EncodeCursor[T]` / `util.DecodeCursor[T]` generic helpers in
`internal/shared/util/cursor.go` — define a typed struct per endpoint.

## Filtering and Sorting

```
GET /vocabulary?topic_slug=food&q=hello&level_ids=1,2
GET /collections?sort=created_at&order=desc     # sort + order as separate params
GET /collections?sort=name&order=asc
```

Validate via `binding:"oneof=..."` on both `sort` and `order`.

## Rate Limiting

Redis-backed token bucket. Headers returned:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640000000
Retry-After: 60  # on 429
```
