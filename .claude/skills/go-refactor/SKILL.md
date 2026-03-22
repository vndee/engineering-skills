---
name: go-refactor
description: Use when refactoring Go code, cleaning up legacy codebases, optimizing performance, or enforcing clean architecture boundaries in a Go/Fiber backend
---

# Go Refactoring & Performance

## Overview

Safe refactoring patterns and performance optimization for Go backends. Always characterize before changing.

**Core principle:** Never refactor without characterization tests. Never optimize without profiling.

## Safe Refactoring Process

1. **Write characterization tests** — capture current behavior
2. **Refactor** — change structure, not behavior
3. **Verify** — all characterization tests still pass
4. **Clean up** — remove temporary tests if redundant

## Architecture Enforcement

**The dependency rule:** Dependencies point inward. Domain imports nothing external.

```
interfaces → application → domain
infrastructure → domain
```

**Violations to check:**
```bash
# Domain should not import infrastructure or interfaces
grep -r "infrastructure\|interfaces\|fiber\|pgx\|sqlalchemy" internal/domain/

# Application should not import infrastructure
grep -r "infrastructure\|pgx\|redis" internal/application/
```

## Common Refactoring Moves

| Smell | Move |
|-------|------|
| Fat handler | Extract use case |
| Duplicate validation | Extract domain value object |
| Concrete dependency | Extract interface + inject |
| Sequential I/O | `errgroup` for concurrent calls |
| God struct | Split by responsibility |
| Copy-paste handlers | Extract middleware |

## Extract Interface

```go
// Before: handler directly uses concrete repo
type Handler struct { repo *postgres.UserRepo }

// After: handler uses interface from domain
type Handler struct { repo domain.UserRepository }
```

## Concurrent I/O with errgroup

```go
g, ctx := errgroup.WithContext(ctx)
var users []domain.User
var posts []domain.Post

g.Go(func() error {
    var err error
    users, err = userRepo.List(ctx)
    return err
})
g.Go(func() error {
    var err error
    posts, err = postRepo.List(ctx)
    return err
})
if err := g.Wait(); err != nil {
    return err
}
```

## Performance Profiling

```go
import _ "net/http/pprof"

// In main.go, add alongside Fiber:
go http.ListenAndServe(":6060", nil)
```

```bash
# CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Memory profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine dump
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

## SQL Optimization

```sql
-- Always check query plans
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@test.com';
```

**N+1 detection:** If you see N queries for N items in logs, fix with JOIN or batch query.

```go
// Bad: N+1
for _, user := range users {
    posts, _ := postRepo.GetByUserID(ctx, user.ID)
}

// Good: batch
posts, _ := postRepo.GetByUserIDs(ctx, userIDs)
```

## pgxpool Tuning

```go
config, _ := pgxpool.ParseConfig(connStr)
config.MaxConns = 25
config.MinConns = 5
config.MaxConnLifetime = 30 * time.Minute
config.MaxConnIdleTime = 5 * time.Minute
```

## Legacy Code Rescue

When working with legacy Go code that has no tests, bad structure, or mixed concerns:

### Step 1: Characterize Before Touching

**Never change legacy code without characterization tests.** Write tests that capture current behavior — even if the behavior is wrong. You need a safety net before refactoring.

```go
// Characterization test: document what the code ACTUALLY does
func TestLegacy_CreateUser_CurrentBehavior(t *testing.T) {
    // This test captures existing behavior, not desired behavior
    // If this test breaks during refactoring, you changed behavior (not just structure)
    result, err := legacyCreateUser(input)
    assert.NoError(t, err)
    assert.Equal(t, expectedOutput, result) // whatever it currently returns
}
```

### Step 2: Identify the Worst Offenders

Prioritize by risk, not by ugliness:
1. **Code handling user input without validation** — security risk, fix first
2. **Code with no error handling** — silent failures, data corruption risk
3. **God files (500+ lines)** — impossible to test, split by responsibility
4. **Circular dependencies** — extract interfaces to break cycles
5. **Dead code** — remove to reduce cognitive load

### Step 3: Incremental Strangler Fig

Don't rewrite — wrap and replace incrementally:

```go
// 1. Extract interface from legacy code
type UserService interface {
    Create(ctx context.Context, input CreateInput) (*User, error)
}

// 2. Legacy implementation stays as-is (for now)
type legacyUserService struct { db *sql.DB }

// 3. New implementation follows clean architecture
type cleanUserService struct { repo domain.UserRepository }

// 4. Feature flag or gradual rollover
func NewUserService(useLegacy bool) UserService {
    if useLegacy { return &legacyUserService{} }
    return &cleanUserService{}
}
```

### Step 4: Add Missing Error Handling

```go
// Before: legacy swallows errors
result, _ := db.Query(query)

// After: handle every error
result, err := db.Query(query)
if err != nil {
    return fmt.Errorf("query users: %w", err)
}
```

### Step 5: Remove Dead Code

```bash
# Find unused functions
grep -rn "^func " --include="*.go" | while read line; do
  func_name=$(echo "$line" | grep -oP 'func \K\w+')
  count=$(grep -rn "$func_name" --include="*.go" | wc -l)
  [ "$count" -le 1 ] && echo "UNUSED: $line"
done
```

**Delete it.** Don't comment it out. Git has history.

## Chains

- **REQUIRED:** Use `superpowers:systematic-debugging` for performance investigation
- **REQUIRED:** Write characterization tests before any refactoring — no exceptions
- **REQUIRED:** Update CLAUDE.md with discovered gotchas and conventions (`claude-md`)
- **Legacy codebases:** Run `fullstack-healthcheck` first to prioritize what to fix
