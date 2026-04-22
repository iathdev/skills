---
name: clean-ddd-hexagonal
description: "DDD + Hexagonal Architecture enforcement for lu-backend (Go/Gin/sqlc/pgx/PostgreSQL). Use when designing domain models, defining aggregates, adding new subdomains, reviewing architecture compliance, or moving business logic into domain entities."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Go/Gin/sqlc/PostgreSQL.
metadata:
  author: thaidong
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(git:*) Agent AskUserQuestion
paths:
  - "internal/**/*.go"
---

# DDD + Hexagonal — Enforcement Rules

You are enforcing DDD tactical patterns and hexagonal architecture for lu-backend.
The project layout and layer structure are already documented in CLAUDE.md — do NOT repeat them.
This skill adds DDD-specific decisions and enforcement on top.

## Enforce These Rules

### Dependency violations — BLOCK these:

- `domain/` importing anything outside stdlib + `uuid`
- Use case importing `gin`, `pgx`, `sqlc`, or HTTP types
- Handler calling repository directly (must go through use case)
- Domain struct with `json:`, `db:`, or framework tags
- Repository returning `AppError` (must return raw errors)
- DTO types passed into domain methods (convert at use case boundary)

### When writing domain entities:

1. Start with **rich domain** — factory constructor + validation + business methods
2. Use `Collection` as the reference implementation (see examples/)
3. Add error sentinels in domain (`errors.New(...)`) — use case maps to `AppError`
4. Entity methods enforce invariants — not just setters
5. Reference other aggregates by **typed ID** (`UserID`, not `*User`, not raw `uuid.UUID`)

### Typed IDs (UserID, CollectionID, VocabularyID, …)

Every aggregate root gets a typed ID so compile-time prevents mix-ups like passing
`UserID` where `CollectionID` is expected. Pattern:

```go
// In the aggregate's domain file
type FeatureID struct {
    uuid.UUID
}

func NewFeatureID() (FeatureID, error) {
    u, err := uuid.NewV7()
    if err != nil {
        return FeatureID{}, err
    }
    return FeatureID{UUID: u}, nil
}

func ParseFeatureID(s string) (FeatureID, error) {
    u, err := uuid.Parse(s)
    if err != nil {
        return FeatureID{}, ErrInvalidFeatureID
    }
    return FeatureID{UUID: u}, nil
}
```

Convention:
- **Embed** `uuid.UUID` so JSON/Marshal/String/comparability come free
- **`New*`** = fresh UUIDv7 (aggregate creation)
- **`Parse*`** = from string at edges (HTTP handler, message consumers)
- Int-based IDs use typedef instead (`type ProficiencyLevelID int16`, `type PrepUserID int64`)
- Add value-object methods for hidden invariants (e.g. `PrepUserID.IsSet() bool` instead of
  comparing to `0` at call sites)

Unwrap at the sqlc boundary only: `id.UUID` when passing to generated params,
`domain.FeatureID{UUID: row.ID}` when mapping rows back.

### When writing repositories:

Repos are **I/O adapters**: accept domain types in, return domain types out, delegate all
logic to the domain/use-case layers. No dedupe, no filtering, no defensive validation.

Transaction propagation uses the package-level helper in
`internal/adapter/repository/tx_manager.go`:

```go
func getQueries(ctx context.Context, queries *db.Queries) *db.Queries {
    if tx := txFromCtx(ctx); tx != nil {
        return queries.WithTx(tx)
    }
    return queries
}
```

Every query call routes through it — `getQueries(ctx, repo.queries).XxxQuery(ctx, ...)` —
so repos automatically join the active transaction when called inside `txManager.Execute`.
Never call `repo.queries.XxxQuery` directly.

### When writing use cases:

1. Call domain factory/methods for business logic — use case orchestrates, doesn't own logic
2. Wrap multi-write operations in `txManager.Execute(ctx, func(ctx) error { ... })`
3. Map domain errors → `AppError` via `mapDomainError()` switch
4. Repo returns `(nil, nil)` for not found → use case returns `apperr.NotFound(key)`
5. Repo returns `(nil, err)` → use case returns `apperr.InternalServerError(key, err)`
6. Ownership check: load entity, call `entity.IsOwnedBy(userID)`, return 404 if false

### When adding a new subdomain:

Follow this exact order (one file per subdomain per layer):

1. `internal/domain/<feature>.go` — entity + error sentinels + methods
2. `internal/application/port/inbound.go` — add use case interface
3. `internal/application/port/outbound.go` — add repository interface
4. `internal/application/dto/<feature>.go` — request/response types
5. `internal/application/usecase/<feature>.go` — orchestration
6. `internal/adapter/repository/query/<feature>.sql` → `make sqlc-generate`
7. `internal/adapter/repository/<feature>.go` — implements repo port
8. `internal/adapter/handler/<feature>.go` — thin: bind → call → respond
9. `internal/app.go` — wire in `NewApp` + register routes in `RegisterRoutes`
10. `internal/shared/i18n/translations/{en,vi}/<feature>.json` — error messages

## Decision Guides

### Entity vs Value Object

| Question | Entity | Value Object |
|----------|--------|-------------|
| Has `uuid.UUID` ID? | Yes | No |
| Equality by identity? | Yes | No — by attributes |
| Persisted independently? | Yes | Embedded/owned |

### Same vs Separate Aggregate

| Question | Same | Separate |
|----------|------|----------|
| Must change atomically? | Same | — |
| Eventually consistent OK? | — | Separate |
| Independent lifecycle? | — | Separate |
| Referenced by ID only? | — | Separate |
| >10 child entities? | — | Split it |

### Anemic → Rich Checklist

When an entity has business logic stuck in its use case, move it:

1. Identify validation/rules in use case that reference only the entity's own fields
2. Create domain method on the entity
3. Add domain error sentinels
4. Use case calls domain method, maps errors via `mapDomainError()`

## Reference Documentation

Load based on what you need — do NOT read all at once:

| File | Load When |
|------|-----------|
| [references/ddd-tactical.md](references/ddd-tactical.md) | Modeling entities, VOs, aggregates, domain services, error sentinels |
| [references/templates.md](references/templates.md) | Quick Go templates for each layer + error flow diagram |
| [references/testing.md](references/testing.md) | Test strategy per layer (domain unit, use case mock, repo integration) |
| [references/compliance.md](references/compliance.md) | 9-category PASS/WARN/FAIL evaluation checklist |
| [examples/collection.md](examples/collection.md) | Collection (rich) + Vocabulary (anemic) full layer walkthrough |
