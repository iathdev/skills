---
name: security-reviewer
description: Security review for Go/Gin/sqlc/pgx/PostgreSQL REST API. Covers secrets management, input validation, SQL injection, auth, error sanitization, rate limiting, security headers, and dependency security. Use when adding endpoints, handling user input, or working with secrets.
---

# Security Review

Security checklist for lu-backend (Go, Gin, sqlc, pgx, PostgreSQL, Redis).

## When to Activate

- Adding new API endpoints
- Handling user input or file uploads
- Working with secrets or credentials
- Implementing authorization logic
- Integrating external APIs

## Pre-Merge Checklist

- [ ] No hardcoded secrets
- [ ] All user input validated (`binding` tags + UUID parsing)
- [ ] No SQL string concatenation (sqlc handles parameterization)
- [ ] Resource ownership checked before mutation
- [ ] No internal errors exposed to client (use AppError)
- [ ] Sensitive data masked in logs
- [ ] Rate limiting on expensive endpoints
- [ ] External calls behind circuit breaker with timeout
- [ ] `govulncheck` clean

## Quick Reference

### Secrets

```go
// FAIL
apiKey := "sk-proj-xxxxx"

// PASS — loaded via Viper from env
cfg.PrepAPIGatewayDomain  // from PREP_API_GATEWAY_DOMAIN
```

### SQL Injection

sqlc generates parameterized queries — SQL injection risk is near-zero **if you don't write raw queries**:

```sql
-- query/vocabulary.sql (sqlc — safe, parameterized)
-- name: GetVocabularyByID :one
SELECT * FROM vocabulary WHERE id = $1;
```

```go
// FAIL — raw pgx with string concat
pool.Query(ctx, "SELECT * FROM users WHERE email = '"+email+"'")

// PASS — raw pgx with placeholder
pool.Query(ctx, "SELECT * FROM users WHERE email = $1", email)
```

### Authorization

```go
// Always check ownership — use domain method
if collection == nil || !collection.IsOwnedBy(userID) {
    return nil, apperr.NotFound("collection.not_found")  // don't leak existence
}
```

### Error Sanitization

```go
// FAIL — leaking internals
c.JSON(500, gin.H{"error": err.Error()})

// PASS — AppError hides cause from client
return apperr.InternalServerError("common.internal_error", err)
```

## Reference Documentation

| File | Content |
|------|---------|
| [references/security-patterns.md](references/security-patterns.md) | Detailed patterns, code examples, and per-category checklists |
