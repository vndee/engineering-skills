---
name: code-quality
description: Use when writing any production code in Go, Python, or React — enforces performance-first patterns, prevents N+1 queries, demands algorithmic efficiency, and ensures security safety at every layer
---

# Code Quality & Performance Standards

## Overview

Every line of code must be written with performance and scalability in mind. No naive implementations. No "fix it later." Write it right the first time.

**Core principle:** Think like a competitive programmer — analyze time/space complexity before writing. O(n) where O(n^2) exists is a bug, not a TODO. Security is not a feature — it's a baseline.

## The Non-Negotiables

### 0. Security Safety — The Baseline

Before performance even matters, the code must be safe:

- **Parameterized queries ALWAYS** — never concatenate user input into SQL. Use `$1, $2` (pgx) or `:param` (SQLAlchemy). No exceptions.
- **Input validation at every boundary** — `ValidateBody[T]` (Go), Pydantic models (Python). Never trust user input.
- **Auth middleware on protected routes** — every endpoint that accesses user data must verify authentication.
- **No hardcoded secrets** — API keys, passwords, JWT secrets come from environment variables.
- **No sensitive data in logs** — never log passwords, tokens, or PII.
- **CORS, security headers, rate limiting** — applied at middleware level from day one.

If security is missing, the code doesn't ship. Period. See `security` skill for full patterns.

These patterns are **never acceptable** in production code:

### 1. N+1 Queries — The Cardinal Sin

```go
// NEVER: N+1 query — O(n) database roundtrips
for _, user := range users {
    posts, _ := postRepo.GetByUserID(ctx, user.ID)
    user.Posts = posts
}

// ALWAYS: Batch query — O(1) database roundtrips
posts, _ := postRepo.GetByUserIDs(ctx, userIDs)
postsByUser := groupBy(posts, func(p Post) uuid.UUID { return p.UserID })
```

```python
# NEVER: N+1 in SQLAlchemy
users = await session.execute(select(UserModel))
for user in users:
    posts = user.posts  # lazy load = N queries

# ALWAYS: Eager load
stmt = select(UserModel).options(selectinload(UserModel.posts))

# Or batch query
stmt = select(PostModel).where(PostModel.user_id.in_(user_ids))
```

### 2. Unbounded Queries

```go
// NEVER: select all rows
rows, _ := pool.Query(ctx, "SELECT * FROM events")

// ALWAYS: paginate with limit/offset, enforce max limit
query := "SELECT * FROM events WHERE org_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3"
```

### 3. Missing Indexes

Every `WHERE`, `JOIN ON`, and `ORDER BY` column in queries that operate on large tables MUST have an index. When adding a query, verify the index exists or create a migration.

```sql
-- Check before shipping
EXPLAIN ANALYZE SELECT * FROM events WHERE user_id = $1 ORDER BY created_at DESC;
-- If you see "Seq Scan" on a large table, add an index
```

### 4. Inefficient Data Structures

```go
// NEVER: O(n) lookup in a loop = O(n*m)
for _, item := range items {
    for _, allowed := range allowedIDs {
        if item.ID == allowed { ... }
    }
}

// ALWAYS: O(1) lookup = O(n)
allowedSet := make(map[uuid.UUID]struct{}, len(allowedIDs))
for _, id := range allowedIDs {
    allowedSet[id] = struct{}{}
}
for _, item := range items {
    if _, ok := allowedSet[item.ID]; ok { ... }
}
```

```python
# NEVER: O(n) membership test
if user_id in user_id_list:  # list = O(n)

# ALWAYS: O(1) membership test
if user_id in user_id_set:  # set = O(1)
```

### 5. Redundant Allocations

```go
// NEVER: allocate in hot loop
for _, item := range items {
    result := make([]byte, 0) // allocation per iteration
    // ...
}

// ALWAYS: pre-allocate with known capacity
results := make([]Response, 0, len(items))
for _, item := range items {
    results = append(results, mapToResponse(item))
}
```

## Database Performance Patterns

### Batch Operations

```go
// NEVER: insert one at a time
for _, user := range users {
    _, err := pool.Exec(ctx, "INSERT INTO users (...) VALUES (...)", ...)
}

// ALWAYS: batch insert
batch := &pgx.Batch{}
for _, user := range users {
    batch.Queue("INSERT INTO users (...) VALUES (...)", ...)
}
results := pool.SendBatch(ctx, batch)
defer results.Close()
```

### Count + Items in One Repo Call

```go
// ALWAYS: repo returns (items, total, error) for list endpoints
func (r *repo) List(ctx context.Context, limit, offset int) ([]Item, int, error) {
    // Count query
    var total int
    err := r.pool.QueryRow(ctx, "SELECT COUNT(*) FROM items WHERE ...").Scan(&total)
    // Items query (same WHERE clause)
    rows, err := r.pool.Query(ctx, "SELECT ... FROM items WHERE ... LIMIT $1 OFFSET $2", limit, offset)
    return items, total, nil
}
```

### Atomic Operations (Transactions)

```go
// When multiple writes must succeed or fail together
tx, _ := pool.Begin(ctx)
defer tx.Rollback(ctx)
// ... multiple operations ...
tx.Commit(ctx)
```

### Query Building

```go
// Dynamic WHERE clauses: build conditions array, use parameterized position args
var conditions []string
var args []interface{}
argPos := 1

conditions = append(conditions, fmt.Sprintf("org_id = $%d", argPos))
args = append(args, orgID)
argPos++

if search != "" {
    conditions = append(conditions, fmt.Sprintf("name ILIKE $%d", argPos))
    args = append(args, "%"+pkg.EscapeLikePattern(search)+"%")
    argPos++
}

whereClause := strings.Join(conditions, " AND ")
```

## Algorithm Standards

Before writing any data processing logic, state the complexity:

| Acceptable | Unacceptable |
|-----------|-------------|
| O(n) single pass | O(n^2) nested loops on same data |
| O(n log n) sort then binary search | O(n) linear search in a loop |
| O(1) hash map lookup | O(n) list scan for membership |
| O(n) with pre-computed map | O(n*m) cross-product without index |
| Streaming/chunked for large data | Loading everything into memory |

## Concurrency Patterns

```go
// When you have independent I/O calls, parallelize
g, ctx := errgroup.WithContext(ctx)
var users []User
var stats Stats

g.Go(func() error {
    var err error
    users, err = userRepo.List(ctx)
    return err
})
g.Go(func() error {
    var err error
    stats, err = statsService.Get(ctx)
    return err
})
if err := g.Wait(); err != nil {
    return err
}
```

```python
# Python: asyncio.gather for independent I/O
users, stats = await asyncio.gather(
    user_repo.list(),
    stats_service.get(),
)
```

## React Performance

- **Virtualize long lists** (>100 items) — use `@tanstack/react-virtual`
- **Debounce search inputs** — don't fire API on every keystroke
- **Paginate API calls** — never fetch unbounded lists
- **Memoize expensive computations** — but only when profiler confirms the cost
- **Code split routes** — `React.lazy` for route-level splitting

## Self-Review Checklist

Before considering any code complete, verify:

- [ ] No N+1 queries — every related data load is batched or joined
- [ ] All list queries have LIMIT — no unbounded selects
- [ ] WHERE/JOIN columns are indexed — checked with EXPLAIN ANALYZE
- [ ] No O(n^2) when O(n) or O(n log n) is possible
- [ ] Pre-allocated slices/arrays with known capacity
- [ ] Independent I/O calls are parallelized (errgroup / asyncio.gather)
- [ ] Batch inserts/updates for multiple rows
- [ ] Transactions for multi-write operations that must be atomic
- [ ] No redundant allocations in hot paths
- [ ] Search queries use proper escaping (LIKE patterns)
