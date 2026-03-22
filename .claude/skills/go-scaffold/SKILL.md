---
name: go-scaffold
description: Use when starting a new Go backend, API, or microservice with Fiber, PostgreSQL, and clean architecture
---

# Go Project Scaffold

## Overview

Bootstrap a new Go/Fiber backend with clean architecture, pgxpool, golang-migrate, Docker, and CI-ready tooling.

**Core principle:** Start with the right structure. Clean architecture from line one.

## Directory Structure

```
project/
  cmd/
    api/
      main.go               # Entry point, wire dependencies
  internal/
    domain/                  # Entities, value objects, repository interfaces
    application/             # Use cases, DTOs, service interfaces
    infrastructure/
      postgres/              # Repository implementations, migrations
      redis/                 # Cache implementations
      config/                # Environment config loading
    interfaces/
      http/                  # Fiber handlers, middleware, routes
        middleware/
        routes.go
  migrations/                # golang-migrate SQL files
  docs/                      # Generated swagger docs (swag init)
  pkg/                       # Shared utilities
    validator.go             # Generic ValidateBody/Query/Params (go-playground/validator)
    response.go              # SuccessResponse, PaginatedResponse, ErrorResponse
  Makefile
  Dockerfile
  docker-compose.yml
  .air.toml
  .golangci.yml
  go.mod
  CLAUDE.md
```

## Bootstrap Files

### main.go
```go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"
    swaggo "github.com/gofiber/contrib/v3/swaggo"
    "github.com/gofiber/fiber/v3"
    _ "project/docs"
)

// @title Project API
// @version 1.0
// @description Project API
// @host localhost:8000
// @BasePath /
// @schemes http
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization

func main() {
    app := fiber.New(fiber.Config{
        ErrorHandler: customErrorHandler,
    })

    // Wire dependencies
    cfg := config.Load()
    pool := setupDatabase(cfg)
    defer pool.Close()
    setupRoutes(app, pool)

    // Swagger docs
    app.Get("/docs/*", swaggo.New(swaggo.Config{}))

    // Graceful shutdown
    go func() {
        if err := app.Listen(":" + cfg.Port); err != nil {
            log.Fatal(err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    app.Shutdown()
}
```

### Makefile
```makefile
.PHONY: run test lint migrate-up migrate-down migrate-create build swagger

run:
	air

test:
	go test ./... -race -count=1

test-integration:
	go test ./... -race -count=1 -tags=integration

lint:
	golangci-lint run

build:
	go build -o bin/api cmd/api/main.go

swagger:
	swag init -g cmd/api/main.go -o docs

migrate-up:
	migrate -path migrations -database "$(DATABASE_URL)" up

migrate-down:
	migrate -path migrations -database "$(DATABASE_URL)" down 1

migrate-create:
	migrate create -ext sql -dir migrations -seq $(name)
```

### docker-compose.yml
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: pg_isready -U app
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

### .air.toml
```toml
[build]
cmd = "go build -o ./tmp/main ./cmd/api"
bin = "tmp/main"
include_ext = ["go"]
exclude_dir = ["tmp", "vendor", "node_modules"]
```

### .golangci.yml
```yaml
linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - unused
    - gosimple
    - ineffassign
    - typecheck
run:
  timeout: 5m
```

## Initial Health Check (TDD)

**REQUIRED:** Invoke `superpowers:test-driven-development` to write `/health` endpoint.

Test first:
```go
func TestHealthCheck(t *testing.T) {
    app := fiber.New()
    app.Get("/health", healthHandler)

    req := httptest.NewRequest("GET", "/health", nil)
    resp, err := app.Test(req)
    require.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)
}
```

## CLAUDE.md Template

```markdown
# Project Name

## Stack
- Go + Fiber
- PostgreSQL (pgxpool) + golang-migrate
- Redis

## Commands
- `make run` — start with hot reload
- `make test` — run unit tests
- `make test-integration` — run integration tests
- `make lint` — run golangci-lint
- `make swagger` — regenerate swagger docs
- `make migrate-up` — apply migrations
- `make migrate-create name=description` — new migration

## Architecture
Clean architecture: domain → application → infrastructure → interfaces
Dependencies point inward. Domain has zero external imports.
```

## Chains

- **REQUIRED:** Update CLAUDE.md with stack, commands, directories, and conventions (`claude-md`)
- Pairs with `react-scaffold` for full-stack setup
- Use `docker-build` for production Dockerfile
- Use `ci-pipeline` for GitHub Actions setup
