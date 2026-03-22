---
name: go-refactor
description: Use when refactoring Go code, optimizing performance, or enforcing clean architecture boundaries in a Go/Fiber backend
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

## Chains

- **REQUIRED:** Use `superpowers:systematic-debugging` for performance investigation
- Write characterization tests before any refactoring
