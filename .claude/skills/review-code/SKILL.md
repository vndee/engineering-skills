---
name: review-code
description: Use when reviewing changed code before committing or after completing a feature — checks performance, architecture, type safety, and code quality against project standards
---

# Code Review

Review all changed files for performance, architecture violations, type safety, and code quality. Fix every real issue found. Severity-ranked — critical issues first.

## Phase 1: Identify Changes & Detect Stack

Run `git diff` (or `git diff HEAD` if there are staged changes) to see what changed. If no git changes, review files edited in this conversation.

Auto-detect stack from changed files:
- `.go` files → Go review rules
- `.py` files → Python review rules
- `.tsx`/`.ts` files → React review rules
- Mixed → apply all relevant rule sets

## Phase 2: Launch Four Review Agents in Parallel

Use the Agent tool to launch all four agents concurrently. Pass each agent the full diff plus the detected stack context.

### Agent 1: Performance & Efficiency Review (CRITICAL)

This is the highest priority review. Check every change against these rules:

**Database (Go/pgx):**
- [ ] **N+1 queries** — any query inside a loop is a bug. Must use batch query, JOIN, or `pgx.Batch`
- [ ] **Unbounded queries** — every SELECT must have LIMIT. No `SELECT * FROM table` without pagination
- [ ] **Missing indexes** — new WHERE/JOIN/ORDER BY columns on large tables need index migrations
- [ ] **Sequential queries** — independent queries that should use `errgroup` for parallel execution
- [ ] **Missing batch inserts** — multiple INSERTs in a loop → use `pgx.Batch` or multi-value INSERT
- [ ] **Inefficient COUNT** — separate COUNT query when it could be `COUNT(*) OVER()` window function

**Database (Python/SQLAlchemy):**
- [ ] **N+1 queries** — lazy-loaded relationships accessed in loops. Must use `selectinload()` or `joinedload()`
- [ ] **Missing eager loading** — related models accessed but not loaded in the query options
- [ ] **Sequential async I/O** — independent awaits that should use `asyncio.gather()`
- [ ] **Unbounded queries** — missing `.limit()` on queries that return lists

**Algorithm & Data Structures (all stacks):**
- [ ] **O(n²) when O(n) possible** — nested loops on same data, list membership in loop (use map/set)
- [ ] **Redundant allocations** — slice/list creation inside loops without pre-allocation
- [ ] **Linear search for lookup** — using list/slice where map/set gives O(1)
- [ ] **String concatenation in loop** — use `strings.Builder` (Go) or `"".join()` (Python)
- [ ] **Unnecessary copies** — passing large structs by value in Go where pointer suffices

**React:**
- [ ] **Unbounded list rendering** — >50 items without virtualization
- [ ] **Missing debounce** — search/filter inputs firing API on every keystroke
- [ ] **Fetch in render** — API calls outside of hooks/React Query
- [ ] **Missing pagination** — fetching all items from API without limit/offset

**General:**
- [ ] **Redundant computation** — same value computed multiple times, repeated file reads, duplicate API calls
- [ ] **Hot-path bloat** — expensive work in startup, per-request, or per-render paths
- [ ] **Memory leaks** — unbounded caches, missing cleanup, event listener leaks
- [ ] **TOCTOU** — checking existence before operating instead of operating and handling error

### Agent 2: Architecture & Structure Review

**Clean architecture boundaries:**
- [ ] **Domain imports external** — `internal/domain/` (Go) or `src/domain/` (Python) must not import infrastructure, interfaces, fiber, pgx, sqlalchemy, fastapi, or any framework
- [ ] **Application imports infrastructure** — `internal/application/` or `src/application/` must not import infrastructure packages directly
- [ ] **Handler contains business logic** — logic beyond parsing request and calling use case belongs in application layer
- [ ] **Domain entity exposed in API response** — handlers must map to DTOs, never return domain types directly
- [ ] **Repository returns non-domain types** — repos must return domain entities, not ORM models

**Go-specific:**
- [ ] **Missing swaggo annotations** — every handler must have `// FuncName godoc` block with @Summary, @Tags, @Param, @Success, @Failure, @Router
- [ ] **Manual body parsing** — must use `pkg.ValidateBody[T]`, `pkg.ValidateQuery[T]`, `pkg.ValidateParams[T]` instead of manual `BodyParser` + validation
- [ ] **Missing `example` tags** — DTO/response structs should have `example` tags for Swagger UI
- [ ] **Error handling** — naked `error` returns without wrapping context. Use `fmt.Errorf("operation: %w", err)`
- [ ] **Exported function without comment** — all exported functions need doc comments

**Python-specific:**
- [ ] **Missing type annotations** — all functions must have return types, all parameters typed
- [ ] **ABC instead of Protocol** — repository interfaces should use `Protocol` for structural typing
- [ ] **Missing `Mapped[]` types** — SQLAlchemy models must use `Mapped[type]` with `mapped_column()`
- [ ] **Untyped dict** — passing `dict` where a Pydantic model or dataclass should be used

**React-specific:**
- [ ] **Prop drilling >3 levels** — should use Context or state library
- [ ] **Business logic in component** — extract to custom hook
- [ ] **Missing loading/error/empty states** — every data-fetching component needs all three

### Agent 3: Code Quality & Reuse Review

**Duplication & reuse:**
- [ ] **Duplicate logic** — search codebase for existing utilities that could replace new code
- [ ] **Copy-paste with variation** — near-duplicate blocks that should be unified
- [ ] **Inline logic for existing util** — hand-rolled string manipulation, path handling, type guards when helpers exist

**Code smells:**
- [ ] **Redundant state** — state that duplicates or can be derived from other state
- [ ] **Parameter sprawl** — function growing too many params instead of using options struct/config
- [ ] **Stringly-typed code** — raw strings where constants/enums already exist in codebase
- [ ] **Magic numbers** — unexplained numeric literals (extract to named constants)
- [ ] **Dead code** — unused functions, unreachable branches, commented-out code

**Error handling:**
- [ ] **Swallowed errors** — `_ = someFunc()` or empty catch blocks hiding failures
- [ ] **Panic in library code** (Go) — use error returns, never panic
- [ ] **Bare except** (Python) — `except Exception` or `except:` without specific error types

### Agent 4: Security & Safety Review (NON-NEGOTIABLE)

**Security is a non-negotiable. Any security finding is automatically CRITICAL.**

- [ ] **SQL injection** — string concatenation in queries instead of parameterized args
- [ ] **Missing input validation** — user input used without validation (should go through ValidateBody/Query/Params or Pydantic)
- [ ] **Sensitive data in logs** — passwords, tokens, API keys logged or included in error messages
- [ ] **Missing auth check** — endpoint without `@Security BearerAuth` annotation or missing auth middleware
- [ ] **CORS misconfiguration** — overly permissive origins in production config
- [ ] **Hardcoded secrets** — API keys, passwords, connection strings in code instead of env vars
- [ ] **Path traversal** — user-controlled file paths without sanitization
- [ ] **Mass assignment** — binding all request fields to model without allowlist
- [ ] **Missing rate limiting** — public endpoints without rate limiting middleware
- [ ] **Insecure defaults** — debug mode, verbose errors, or stack traces exposed in production config

## Phase 3: Aggregate & Fix

Wait for all four agents to complete. Aggregate findings by severity:

### Severity Levels

| Level | Definition | Action |
|-------|-----------|--------|
| **CRITICAL** | N+1 queries, SQL injection, unbounded queries, security holes, architecture violations | Must fix before commit |
| **HIGH** | Missing indexes, O(n²) algorithms, missing validation, swallowed errors | Must fix before commit |
| **MEDIUM** | Missing swaggo annotations, type safety gaps, code duplication, missing error context | Fix now, acceptable to defer with TODO |
| **LOW** | Missing example tags, minor style issues, potential future optimization | Note and skip if time-constrained |

### Fix Process

1. Fix all CRITICAL and HIGH issues directly in code
2. Fix MEDIUM issues — skip only if clearly a false positive
3. Note LOW issues in summary — do not spend time on these
4. Do NOT argue with findings or add defensive commentary — fix or skip

### Output Format

```
## Code Review Results

### Fixed (N issues)
- [CRITICAL] Fixed N+1 query in UserRepository.List — batch query with IN clause
- [HIGH] Added LIMIT to events query in ActivityService
- ...

### Skipped (N issues)
- [LOW] Missing example tag on InternalDTO — not exposed in Swagger
- ...

### Clean
- No architecture violations found
- Security review passed
```

## Chains

- **Invoked by:** `eng-lead` (code review gate before committing)
- **Complements:** All `*-feature` and `*-refactor` skills as final quality gate
- **REQUIRED:** Fix all CRITICAL and HIGH issues before code is committed
- **REQUIRED:** Update CLAUDE.md if review reveals project-specific gotchas (`claude-md`)
