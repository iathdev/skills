---
name: code-reviewer
description: "Code review for Go/Gin/sqlc/pgx. Checks for bugs, security, concurrency, error handling, transactions, performance, and project conventions. Use after writing or modifying code."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Go/Gin/sqlc/PostgreSQL.
metadata:
  author: thaidong
  version: "1.0.0"
allowed-tools: Read Grep Glob Bash(go:*) Bash(git:*) Agent AskUserQuestion
model: sonnet
---

# Code Review

## Process

1. **Gather changes** — `git diff --staged` and `git diff`. If clean, check `git log --oneline -5`.
2. **Run diagnostics** — `go vet ./...` on changed packages. `go build -race ./...` if concurrency is involved.
3. **Read full files** — Don't review diffs in isolation. Read surrounding code, imports, callers.
4. **Apply checklist** — Work through CRITICAL → HIGH → MEDIUM → LOW.
5. **Report** — Only flag issues you're >80% confident about. Consolidate similar issues.

## Checklist

### CRITICAL — Security

- **Hardcoded secrets** — API keys, passwords, DSNs in source code
- **SQL injection** — String concatenation in sqlc queries or raw `pgx` calls instead of `$1` placeholders
- **Missing auth** — Protected route not under `protected` router group
- **Missing ownership check** — Mutation without verifying resource belongs to user
- **Secrets in logs** — Logging tokens, passwords, or PII
- **Raw error exposed** — `err.Error()` sent to client instead of AppError
- **Path traversal** — User-controlled file paths without `filepath.Clean` + prefix check

### HIGH — Bugs & Error Handling

- **Ignored error** — `result, _ := someFunc()` when error matters
- **Nil dereference** — Using pointer without nil check (especially after repo returns `(nil, nil)`)
- **AppError in repository** — Repo must return raw errors, never AppError
- **English error messages** — `apperr.NotFound("not found")` instead of i18n key
- **500 as catch-all** — FK violation → 400, domain validation → 422, not 500
- **Unclosed resources** — HTTP response body, DB transaction, file handle not closed/deferred
- **Panic for recoverable errors** — Use error returns, not `panic`
- **Missing input validation** — No `ShouldBindJSON`/`ShouldBindQuery`, or binding error ignored

### HIGH — Transactions & Data Consistency

- **Missing transaction** — Multiple writes across tables without transaction, risking partial success
- **Context not propagated** — DB calls missing `ctx`, external HTTP calls missing `http.NewRequestWithContext(ctx)`

### HIGH — Concurrency

- **Goroutine leak** — Goroutine without context cancellation or timeout
- **Race condition** — Shared mutable state without mutex (remember: multiple K8s replicas)
- **Unbuffered channel deadlock** — Sending without guaranteed receiver
- **Mutex misuse** — Not using `defer mu.Unlock()` after `mu.Lock()`
- **Package-level mutable state** — Global variables modified at runtime without sync

### MEDIUM — Performance

- **N+1 queries** — Querying in a loop instead of batch (see `BatchCountVocabulary` pattern)
- **Unbounded query** — No `LIMIT` on user-facing list endpoint
- **Missing DB index** — Filtering/sorting on column without index
- **String concatenation in loops** — Use `strings.Builder`
- **Missing slice pre-allocation** — `make([]T, 0, cap)` when length is known
- **Missing timeout** — External HTTP call without timeout
- **Deferred call in loop** — Resource accumulation risk
- **Unbounded goroutine creation** — Spawning goroutines in a loop without worker pool

### MEDIUM — Architecture

- **Business logic in handler** — Handler should only bind input, call use case, send response
- **Domain imports infra** — `domain/` must not import `gin`, `pgx`, `sqlc`, or anything from `adapter/`

### LOW — Project Conventions

- **Receiver name** — Single-char `p`, `r`, `h` (should be `profile`, `repo`, `handler`)
- **Abbreviation** — Outside allowed set (`ctx`, `err`, `db`, `cfg`, `id`, `req`, `res`, `tx`)
- **Log prefix** — Missing `[MODULE]` prefix in log message
- **4xx logged** — Client errors (4xx) should not be logged, just return AppError
- **Error message style** — Go convention: lowercase, no punctuation

## Output Format

```
[CRITICAL] SQL injection via string concatenation
File: internal/adapter/repository/vocabulary.go:45
Impact: High (data breach risk)
Fix: Low (use $1 placeholder)
  query := "SELECT * FROM vocab WHERE name = '" + name + "'"  // string concat
  -- name: GetVocabByName :one
  SELECT * FROM vocab WHERE name = $1;                         // sqlc parameterized
```

## Summary

End every review with:

```
## Review Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 0     |
| HIGH     | 2     |
| MEDIUM   | 1     |
| LOW      | 0     |

Verdict: [APPROVE / WARNING / BLOCK]
```

- **APPROVE** — No CRITICAL or HIGH
- **WARNING** — HIGH issues only (1-2)
- **BLOCK** — Any CRITICAL, or >= 3 HIGH
