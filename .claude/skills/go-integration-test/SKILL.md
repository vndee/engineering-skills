---
name: go-integration-test
description: Use when writing integration tests for Go services that need real PostgreSQL, Redis, or other external dependencies via testcontainers
---

# Go Integration Testing

## Overview

Integration tests with real dependencies using testcontainers-go. No mocks for infrastructure — test against actual Postgres and Redis.

**Core principle:** If it talks to a database, test it with a real database.

## When to Use

- Testing repository implementations against real Postgres
- Testing Redis cache/session logic
- Full HTTP lifecycle tests (handler → use case → repo → DB)
- Verifying migration correctness

## Testcontainers Setup

```go
//go:build integration

package postgres_test

import (
    "context"
    "testing"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
)

func setupPostgres(t *testing.T) *pgxpool.Pool {
    t.Helper()
    ctx := context.Background()

    container, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2)),
    )
    require.NoError(t, err)
    t.Cleanup(func() { container.Terminate(ctx) })

    connStr, err := container.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    pool, err := pgxpool.New(ctx, connStr)
    require.NoError(t, err)
    t.Cleanup(pool.Close)

    // Run migrations
    runMigrations(t, connStr)
    return pool
}
```

## Running Migrations in Tests

```go
func runMigrations(t *testing.T, connStr string) {
    t.Helper()
    m, err := migrate.New("file://../../migrations", connStr)
    require.NoError(t, err)
    err = m.Up()
    require.NoError(t, err)
}
```

## Test Isolation

```go
// Option 1: Transaction rollback (fastest)
func withTx(t *testing.T, pool *pgxpool.Pool) pgx.Tx {
    t.Helper()
    tx, err := pool.Begin(context.Background())
    require.NoError(t, err)
    t.Cleanup(func() { tx.Rollback(context.Background()) })
    return tx
}

// Option 2: Truncate tables between tests
func cleanTables(t *testing.T, pool *pgxpool.Pool, tables ...string) {
    t.Helper()
    for _, table := range tables {
        _, err := pool.Exec(context.Background(),
            "TRUNCATE TABLE "+table+" CASCADE")
        require.NoError(t, err)
    }
}
```

## Repository Test Example

```go
func TestUserRepository_Create(t *testing.T) {
    pool := setupPostgres(t)
    repo := postgres.NewUserRepository(pool)

    user, err := domain.NewUser("test@example.com", "Test User")
    require.NoError(t, err)

    err = repo.Create(context.Background(), user)
    require.NoError(t, err)

    found, err := repo.GetByID(context.Background(), user.ID)
    require.NoError(t, err)
    assert.Equal(t, user.Email, found.Email)
}
```

## Full HTTP Lifecycle Test

```go
func TestCreateUser_Integration(t *testing.T) {
    pool := setupPostgres(t)
    app := setupApp(pool) // wire up fiber app with real deps

    body := `{"email":"test@example.com","name":"Test"}`
    req := httptest.NewRequest("POST", "/api/v1/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    resp, err := app.Test(req)
    require.NoError(t, err)
    assert.Equal(t, 201, resp.StatusCode)
}
```

## Build Tags & Makefile

```makefile
test:
	go test ./... -race -count=1

test-integration:
	go test ./... -race -count=1 -tags=integration

test-all:
	go test ./... -race -count=1 -tags=integration
```

## Common Mistakes

- Forgetting `//go:build integration` tag — tests run in CI without Docker
- Sharing containers across tests without isolation — data leaks
- Not using `t.Parallel()` carefully — shared state conflicts
- Hardcoding ports — let testcontainers pick random ports
