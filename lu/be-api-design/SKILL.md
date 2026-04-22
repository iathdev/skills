---
name: api-design
description: "REST API design patterns for lu-backend — endpoint design, response envelope, error handling, pagination, validation, handler/use-case/repository conventions following hexagonal architecture with Gin/sqlc/PostgreSQL. Use when designing new endpoints, reviewing API contracts, or adding pagination/filtering/sorting."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Go/Gin/sqlc/PostgreSQL.
metadata:
  author: thaidong
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(git:*) Agent AskUserQuestion
---

# API Design Patterns — lu-backend

Go + Gin + sqlc + PostgreSQL, flat hexagonal architecture.

## When to Activate

- Designing new API endpoints
- Reviewing existing API contracts
- Adding pagination, filtering, or sorting
- Updating OpenAPI spec (`docs/openapi.yaml`)

## URL Rules

Resources are **nouns, plural, lowercase, kebab-case**.

```
POST   /api/v1/<resources>
GET    /api/v1/<resources>
GET    /api/v1/<resources>/:id
PATCH  /api/v1/<resources>/:id
DELETE /api/v1/<resources>/:id
GET    /api/v1/<resources>/:id/<sub-resources>
POST   /api/v1/<resources>/:id/<sub-resources>
```

```
# GOOD                          # BAD
/vocabularies                   /getVocabularies (verb in URL)
/grammar-points                 /vocabulary (singular)
/vocabularies?q=hello           /grammar_points (snake_case)
```

## Route Registration

Routes registered in `internal/app.go` via `App.RegisterRoutes(public, protected)`:

```go
func (app *App) RegisterRoutes(public, protected *gin.RouterGroup) {
    v1Public := public.Group("/v1")
    v1Public.GET("/vocabulary", app.vocabHandler.ListVocabulary)

    v1Protected := protected.Group("/v1")
    v1Protected.POST("/collections", app.collectionHandler.CreateCollection)
}
```

## Status Codes

```
200 OK        — GET, PATCH, DELETE
201 Created   — POST
400 BadRequest       — malformed input, FK violation
401 Unauthorized     — missing/invalid token
403 Forbidden        — authenticated but not allowed
404 NotFound         — resource doesn't exist
409 Conflict         — duplicate
422 ValidationFailed — field-level validation errors
429 TooManyRequests  — rate limit
500 InternalServerError — unexpected system errors only
503 ServiceUnavailable  — external service down, circuit breaker open
```

## Handler Pattern

Thin adapter: **bind → call → respond**. Zero business logic.

```go
func (handler *CollectionHandler) CreateCollection(c *gin.Context) {
    userID := c.MustGet("user_id").(domain.UserID)

    name := strings.TrimSpace(c.PostForm("name"))
    if name == "" {
        response.ValidationError(c, nil)
        return
    }

    res, err := handler.useCase.CreateCollection(c.Request.Context(), userID, name)
    if err != nil {
        response.HandleError(c, err)
        return
    }
    response.Success(c, http.StatusCreated, res)
}
```

`c.MustGet("user_id").(domain.UserID)` panics if the middleware invariant is broken —
`AuthMiddleware` always sets the typed `UserID` before handlers run, and
`RecoveryMiddleware` turns any panic into a 500 with a sanitized response.

For path-param aggregate IDs, delegate validity to the domain factory:

```go
func parseCollectionID(c *gin.Context) (domain.CollectionID, bool) {
    id, err := domain.ParseCollectionID(c.Param("collection_id"))
    if err != nil {
        response.NotFound(c, "collection.not_found")
        return domain.CollectionID{}, false
    }
    return id, true
}
```

## Pagination

Two variants, discriminated by `meta.pagination.type`:

- **Offset** (`page`, `page_size`) — CRUD lists with total count
- **Cursor** (`cursor`, `limit`) — feeds, large datasets, real-time data

## Design Checklist

- [ ] URL: plural, kebab-case, no verbs
- [ ] Correct HTTP method and status code
- [ ] Response uses unified envelope (`response.Success` / `response.HandleError`)
- [ ] Error messages are i18n keys, not English
- [ ] List endpoints have pagination
- [ ] Slice inputs (e.g. `[]uuid.UUID`) have `binding:"...,unique,dive"` to reject duplicates at 422
- [ ] Aggregate IDs in path go through `domain.ParseXxxID` (not `uuid.Parse`)
- [ ] API request/response DTOs use **primitive** types (`uuid.UUID`, `int16`, `int64`) — domain wrappers stay inside the hexagon. Usecase unwraps to primitives at the response boundary.
- [ ] Internal filter/criteria structs (usecase ↔ repo) MAY use domain typed IDs — they don't serialize
- [ ] Route on `protected` or `public` group intentionally
- [ ] OpenAPI spec updated (`docs/openapi.yaml`) — list inputs marked `uniqueItems: true`

## Reference Documentation

| File | Content |
|------|---------|
| [references/response-and-errors.md](references/response-and-errors.md) | Response envelope, error object, AppError constructors, repo error convention, pagination specs |
