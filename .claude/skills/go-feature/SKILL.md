---
name: go-feature
description: Use when implementing a new feature, endpoint, or use case in a Go/Fiber backend with clean architecture and PostgreSQL
---

# Go Feature Development

## Overview

Layer-by-layer TDD workflow for Go/Fiber backends following clean architecture. Build inside-out: domain → application → infrastructure → interfaces.

**Core principle:** Every layer gets its own test before implementation. Never skip a layer.

## When to Use

- Adding a new API endpoint to a Go/Fiber service
- Implementing a new use case or business logic
- Adding CRUD operations with PostgreSQL
- Building domain entities with validation

## Layer-by-Layer Workflow

### 1. Domain Entity + Validation

```go
// internal/domain/user.go
type User struct {
    ID        uuid.UUID
    Email     string
    Name      string
    CreatedAt time.Time
}

func NewUser(email, name string) (*User, error) {
    if email == "" {
        return nil, ErrEmptyEmail
    }
    // validation...
}
```

**Test first:** Table-driven tests for all validation rules.

```go
func TestNewUser(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr error
    }{
        {"valid", "a@b.com", nil},
        {"empty email", "", ErrEmptyEmail},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := NewUser(tt.email, "name")
            assert.ErrorIs(t, err, tt.wantErr)
        })
    }
}
```

### 2. Repository Interface (in domain)

```go
// internal/domain/user_repository.go
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id uuid.UUID) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
}
```

### 3. Use Case + Mock Tests

```go
// internal/application/create_user.go
type CreateUserUseCase struct {
    repo domain.UserRepository
}

func (uc *CreateUserUseCase) Execute(ctx context.Context, input CreateUserInput) (*domain.User, error) {
    // validate, create entity, persist
}
```

**Test with mocks:** Use `testify/mock` or manual mocks for the repository interface.

### 4. PostgreSQL Repository + Integration Test

```go
// internal/infrastructure/postgres/user_repository.go
type userRepository struct {
    pool *pgxpool.Pool
}
```

**Test with testcontainers:** Real Postgres, real queries.
**REQUIRED:** Invoke `go-integration-test` for testcontainers patterns.

### 5. Fiber Handler + HTTP Test

```go
// internal/interfaces/http/user_handler.go

// CreateUser godoc
// @Summary Create a new user
// @Description Create a new user account
// @Tags users
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param request body dto.CreateUserRequest true "Create user request"
// @Success 201 {object} dto.UserResponse
// @Failure 400 {object} pkg.ErrorResponse "Validation error"
// @Failure 401 {object} pkg.ErrorResponse "Unauthorized"
// @Failure 500 {object} pkg.ErrorResponse "Internal server error"
// @Router /api/v1/users [post]
func (h *UserHandler) Create(c fiber.Ctx) error {
    body, err := pkg.ValidateBody[CreateUserRequest](c)
    if err != nil {
        return err // returns structured ErrorResponse automatically
    }
    result, err := h.useCase.Execute(c.Context(), body.ToDomain())
    if err != nil {
        return mapDomainError(err)
    }
    return c.Status(201).JSON(result.ToResponse())
}
```

**Every handler MUST have swaggo annotations.** Add `example` tags to DTO structs. Run `swag init -g cmd/api/main.go -o docs` after adding/changing handlers. See `api-design` for full annotation reference.

Use generic `pkg.ValidateBody[T]`, `pkg.ValidateQuery[T]`, `pkg.ValidateParams[T]` — never manually parse + validate. Validation uses struct tags (`validate:"required,email"`) with `go-playground/validator/v10`. Supports `default` tags for zero-value defaults.

**Test:** Use `app.Test(httptest.NewRequest(...))` for HTTP-level tests.

### 6. Route Registration

```go
api := app.Group("/api/v1")
users := api.Group("/users")
users.Post("/", userHandler.Create)
users.Get("/:id", userHandler.GetByID)
```

## Quick Reference

| Layer | Package | Tests | Dependencies |
|-------|---------|-------|-------------|
| Domain | `internal/domain/` | Unit (table-driven) | None |
| Application | `internal/application/` | Unit (mocked repo) | Domain |
| Infrastructure | `internal/infrastructure/` | Integration (testcontainers) | Domain |
| Interfaces | `internal/interfaces/http/` | HTTP (fiber test) | Application |

## Common Patterns

- **DTOs:** Request/Response structs in interfaces layer, never expose domain directly
- **Errors:** Domain errors → HTTP status mapping in handler middleware
- **Pagination:** Offset-based (`limit`/`offset`), repo returns `(items, total, error)`, handler wraps in `PaginatedResponse`
- **Transactions:** `UnitOfWork` interface in domain, implemented in infrastructure

## Chains

- **REQUIRED:** Invoke `superpowers:test-driven-development` at each layer
- **REQUIRED:** Follow `code-quality` standards — no N+1 queries, batch operations, pre-allocated slices, indexed columns, O(n) not O(n^2)
- **REQUIRED:** Update CLAUDE.md if new conventions, commands, or patterns are introduced (`claude-md`)
- **Schema changes:** Invoke `db-migrate` before implementing repository
- **API conventions:** Reference `api-design` for request/response patterns
