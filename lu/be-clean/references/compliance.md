# Architecture Compliance Evaluation

Use this checklist to evaluate whether code changes comply with DDD + Clean Architecture + Hexagonal patterns in lu-backend.

## Scoring

- **PASS** — Fully compliant
- **WARN** — Minor deviation, acceptable with justification
- **FAIL** — Architectural violation, must fix

---

## 1. Dependency Rule

| Check | Expected | Severity |
|-------|----------|----------|
| `domain/` imports only stdlib + `uuid` | No `gin`, `gorm`, `pgx`, `sqlc`, `application/`, `adapter/`, `shared/` | FAIL |
| `application/usecase/` imports no adapter types | No `gin.Context`, `db.Queries`, `pgx` | FAIL |
| `application/port/` uses domain types + DTOs only | No framework types in port signatures | FAIL |
| Handler never imports repository | Handler → use case → repository | FAIL |
| Domain types have no struct tags | No `json:`, `gorm:`, `db:` tags | WARN |

### How to Check

```bash
# Verify domain has no forbidden imports
grep -r "import" internal/domain/ | grep -E "(gin|gorm|pgx|application|adapter|shared|infra)"
# Should return nothing

# Verify use case has no adapter imports
grep -r "import" internal/application/usecase/ | grep -E "(gin|gorm|pgx|adapter)"
# Should return nothing
```

---

## 2. Port Conventions

| Check | Expected | Severity |
|-------|----------|----------|
| All use case ports in `port/inbound.go` | Not scattered across files | WARN |
| All repo/service ports in `port/outbound.go` | Not scattered across files | WARN |
| Inbound port naming: `{Subdomain}UseCasePort` | Consistent naming | WARN |
| Outbound port naming: `{Aggregate}RepositoryPort` or `{Service}ServicePort` | Consistent naming | WARN |
| Port methods use `context.Context` as first param | Required for all I/O operations | FAIL |
| Port returns domain types or DTOs | Never returns DB models or framework types | FAIL |

---

## 3. Aggregate Design

| Check | Expected | Severity |
|-------|----------|----------|
| One repository port per aggregate root | Not per table or per entity | WARN |
| Cross-aggregate references by typed ID | `UserID`, not `*User`, not raw `uuid.UUID` | FAIL |
| Aggregate root has typed ID (`FooID`) | Embed `uuid.UUID` + `NewFooID` + `ParseFooID` | WARN |
| Aggregate root controls child mutations | No direct child access from outside | WARN |
| Aggregate has <10 child entity types | Split if larger | WARN |
| Business rules live in domain entities | Not all logic in use cases (anemic model) | WARN |
| Hidden invariants expressed as value-object methods | `PrepUserID.IsSet()` not `id != 0` at call sites | WARN |

---

## 4. Error Handling

| Check | Expected | Severity |
|-------|----------|----------|
| Repository returns raw errors, never `AppError` | `(nil, nil)` for not found, `(nil, err)` for DB error | FAIL |
| Use case wraps errors into `AppError` | Uses `apperr.NotFound()`, `apperr.InternalServerError()` etc. | FAIL |
| Error messages are i18n keys | `"feature.not_found"`, not `"Not found"` | FAIL |
| Handler only calls `response.HandleError(ctx, err)` | No manual error mapping in handler | WARN |
| 5xx errors carry cause | `apperr.InternalServerError(key, err)` | FAIL |
| 4xx errors have no cause | `apperr.NotFound(key)` — no second arg | WARN |

---

## 5. Handler Layer

| Check | Expected | Severity |
|-------|----------|----------|
| Handler is thin (bind → call → respond) | No business logic | FAIL |
| Handler binds input to DTO | Uses `ShouldBindJSON`, `ShouldBindQuery` | WARN |
| Handler extracts context values | `c.MustGet("user_id").(domain.UserID)`, `c.GetString("lang")` | WARN |
| Handler makes single use case call | Not multiple repo calls | FAIL |
| Handler uses `response.Success()` / `response.HandleError()` | Unified envelope | FAIL |
| No receiver name `h` or `c` for handler | Use `handler` | WARN |

---

## 6. Repository Layer

| Check | Expected | Severity |
|-------|----------|----------|
| Repository wraps sqlc queries | Uses `db.Queries` struct | WARN |
| Maps DB model → domain entity | Never returns `db.XxxRow` directly | FAIL |
| Not found → `(nil, nil)` | Checks `pgx.ErrNoRows` | FAIL |
| Error → `(nil, err)` raw | Never wraps in `AppError` | FAIL |
| No receiver name `r` | Use `repo` | WARN |

---

## 7. Composition Root

| Check | Expected | Severity |
|-------|----------|----------|
| All deps wired in `app.go` `NewApp()` | No DI framework, no global vars | WARN |
| Wiring order: repos → use cases → handlers | Dependencies created before dependents | WARN |
| Routes registered in `RegisterRoutes()` | Not scattered | WARN |

---

## 8. Naming Conventions

| Check | Expected | Severity |
|-------|----------|----------|
| No single-char receivers | `handler`, `repo`, `useCase` — not `h`, `r`, `u` | WARN |
| No abbreviations except allowed | Only: `ctx`, `err`, `db`, `cfg`, `id`, `req`, `res`, `tx` | WARN |
| Log messages have module prefix | `[AUTH]`, `[VOCABULARY]`, `[COLLECTION]` | WARN |
| File naming: one file per subdomain per layer | `handler/collection.go`, `usecase/collection.go` | WARN |

---

## 9. DTO Discipline

| Check | Expected | Severity |
|-------|----------|----------|
| API never exposes domain types directly | Always map domain → DTO | WARN |
| DTOs have `json:` tags | For request/response serialization | WARN |
| Request DTOs have `binding:` tags | For Gin validation | WARN |
| Domain types have NO `json:` tags | Separate concerns | WARN |
| DTO ≠ DB model | sqlc-generated types stay in `adapter/repository/db/` | FAIL |

---

## Evaluation Template

When reviewing code, use this format:

```
## Architecture Compliance Review

### Dependency Rule
- [ ] PASS / WARN / FAIL — (details)

### Port Conventions
- [ ] PASS / WARN / FAIL — (details)

### Aggregate Design
- [ ] PASS / WARN / FAIL — (details)

### Error Handling
- [ ] PASS / WARN / FAIL — (details)

### Handler Layer
- [ ] PASS / WARN / FAIL — (details)

### Repository Layer
- [ ] PASS / WARN / FAIL — (details)

### Naming Conventions
- [ ] PASS / WARN / FAIL — (details)

### Overall: PASS / WARN / FAIL
```
