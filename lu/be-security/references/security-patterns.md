# Security Patterns — Detailed Reference

## 1. Secrets Management

### Checklist

- [ ] No hardcoded secrets in source code
- [ ] All secrets in environment variables (`.env` for local, platform config for prod)
- [ ] `.env` in `.gitignore`
- [ ] No secrets in git history (`git log -p --all -S 'password'`)

## 2. Input Validation

```go
type CreateRequest struct {
    Name  string `json:"name" binding:"required,min=1,max=100"`
    Email string `json:"email" binding:"required,email"`
}

var req dto.CreateRequest
if err := c.ShouldBindJSON(&req); err != nil {
    response.ValidationError(c, err)  // 422 with field-level details
    return
}
```

URL params — always validate via the domain factory so the aggregate's own validity
rule is the single source of truth:

```go
id, err := domain.ParseVocabularyID(c.Param("vocabulary_id"))
if err != nil {
    response.NotFound(c, "vocabulary.not_found")  // don't leak format details
    return
}
```

For slice inputs (JSON body), use `binding:"...,unique,dive"` so duplicates reject
at 422 before reaching the repo, where they may cause `ON CONFLICT` anomalies:

```go
type CollectionVocabularyIdsRequest struct {
    VocabularyIDs []uuid.UUID `json:"vocabulary_ids" binding:"required,min=1,max=50,unique,dive"`
}
```

### Checklist

- [ ] All request bodies validated via `binding` tags
- [ ] URL params parsed and validated (UUID, int)
- [ ] Query params bound via `ShouldBindQuery` with constraints
- [ ] Whitelist validation (not blacklist)

## 3. SQL Injection Prevention

sqlc generates type-safe, parameterized queries. Risk surfaces only from raw `pgx` calls:

```go
// FAIL — raw pgx with string concat
pool.Query(ctx, "SELECT * FROM users WHERE email = '"+email+"'")
pool.Query(ctx, fmt.Sprintf("SELECT * FROM users WHERE name LIKE '%%%s%%'", name))

// PASS — always use placeholders
pool.Query(ctx, "SELECT * FROM users WHERE email = $1", email)
pool.Query(ctx, "SELECT * FROM users WHERE name LIKE '%' || $1 || '%'", name)
```

### Checklist

- [ ] All queries via sqlc-generated functions (preferred) or parameterized `$1` placeholders
- [ ] No string concatenation in SQL
- [ ] `LIKE` patterns: user input passed via placeholder, not interpolated

## 4. Authentication & Authorization

Auth flow: Bearer token → `AuthMiddleware` → `App.Authenticate()` → stores `user_id` in context.

Ownership check pattern (from collection use case):

```go
func (useCase *CollectionUseCase) loadOwnedCollection(ctx context.Context, userID, collectionID uuid.UUID) (*domain.Collection, error) {
    collection, err := useCase.collectionRepo.FindByID(ctx, collectionID)
    if err != nil {
        return nil, apperr.InternalServerError("common.internal_error", err)
    }
    // Return 404 for both not-found AND not-owned (don't leak existence)
    if collection == nil || !collection.IsOwnedBy(userID) {
        return nil, apperr.NotFound("collection.not_found")
    }
    return collection, nil
}
```

### Checklist

- [ ] Every protected endpoint under `protected` router group
- [ ] Resource ownership verified before mutation
- [ ] Token not logged (masked in request logger)

## 5. Error Sanitization

- 4xx `AppError`: no cause attached, client gets translated i18n key
- 5xx `AppError`: cause attached for server-side logging, client gets generic message
- Panic recovery: returns `common.internal_error`, stack trace → Sentry

### Checklist

- [ ] No `err.Error()` exposed to client
- [ ] No stack traces in responses
- [ ] 5xx errors return generic messages
- [ ] Detailed errors logged server-side only

## 6. Sensitive Data in Logs

Request logger masks these keys automatically: `password`, `token`, `secret`, `authorization`, `cookie`, `credit_card`, `ssn`, `api_key` → replaced with `***`.

Tokens are SHA256-hashed before use as Redis cache keys.

### Checklist

- [ ] No passwords, tokens, or secrets in log output
- [ ] New sensitive fields added to mask list if needed
- [ ] 4xx → don't log; 5xx → log with context

## 7. Rate Limiting

Already implemented:
- **Global**: Redis-backed token bucket on all public routes (per-IP)
- **Per-route**: Individual limits (e.g., `/health` at 10 req/min)
- **Distributed**: Lua scripts for atomic operations across K8s replicas

Adding rate limit to a new route:

```go
r.POST("/api/v1/ocr/scan",
    middleware.RateLimitMiddleware(redisClient, 5),  // 5 req/min per IP
    handler.Scan,
)
```

## 8. Security Headers

Already implemented in `SecurityHeadersMiddleware`:

| Header | Value |
|--------|-------|
| X-Content-Type-Options | `nosniff` |
| X-Frame-Options | `DENY` |
| Referrer-Policy | `strict-origin-when-cross-origin` |
| Strict-Transport-Security | `max-age=31536000; includeSubDomains` |
| Content-Security-Policy | `default-src 'self'` |

## 9. External API Calls

Pattern from PrepUserService:

```go
result, err := breaker.Execute(func() (*domain.PrepUser, error) {
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
    req.Header.Set("Authorization", "Bearer "+token)
    resp, err := client.Do(req)
    // ...
})
```

### Checklist

- [ ] External calls wrapped in circuit breaker
- [ ] Timeout configured
- [ ] Errors mapped to `apperr.ServiceUnavailable`
- [ ] No raw external error messages exposed to client

## 10. Dependencies

```bash
govulncheck ./...   # check vulnerabilities
go mod verify       # verify checksums
```

- [ ] `go.sum` committed
- [ ] No known vulnerabilities
- [ ] No `replace` directives pointing to local paths
