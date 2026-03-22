---
name: api-design
description: Use when designing new API endpoints, reviewing API consistency, or establishing REST conventions for Go/Fiber or Python/FastAPI backends
---

# API Design Patterns

## Overview

Consistent REST API conventions for Go/Fiber and Python/FastAPI backends. Covers URL structure, response envelopes, error handling, and middleware.

**Core principle:** APIs should be predictable. Same patterns everywhere, every time.

## When to Use

- Designing new endpoints
- Reviewing API consistency across services
- Setting up middleware ordering
- Establishing error response formats

## URL Conventions

```
GET    /api/v1/resources          # List (paginated)
POST   /api/v1/resources          # Create
GET    /api/v1/resources/:id      # Get by ID
PUT    /api/v1/resources/:id      # Full update
PATCH  /api/v1/resources/:id      # Partial update
DELETE /api/v1/resources/:id      # Delete

# Nested resources (max 1 level deep)
GET    /api/v1/users/:id/posts    # User's posts
POST   /api/v1/users/:id/posts    # Create post for user

# Actions (when CRUD doesn't fit)
POST   /api/v1/resources/:id/archive
POST   /api/v1/resources/:id/send
```

## Response Envelope

```json
// Success
{
    "success": true,
    "message": "Operation completed successfully"
}

// Success (paginated list)
{
    "items": [ ... ],
    "pagination": {
        "limit": 20,
        "offset": 0,
        "total": 100
    }
}

// Error
{
    "code": "validation_error",
    "message": "Validation failed",
    "details": [
        { "field": "email", "message": "email is required" }
    ]
}
```

## Go Response Types

```go
// pkg/response.go
type SuccessResponse struct {
    Success bool   `json:"success"`
    Message string `json:"message"`
}

type PaginationResponse struct {
    Limit  int `json:"limit"`
    Offset int `json:"offset"`
    Total  int `json:"total"`
}

type PaginatedResponse struct {
    Items      interface{}        `json:"items"`
    Pagination PaginationResponse `json:"pagination"`
}

type ErrorResponse struct {
    StatusCode int          `json:"-"`
    Code       string       `json:"code"`
    Message    string       `json:"message"`
    Details    []FieldError `json:"details,omitempty"`
}

type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}
```

## HTTP Status Mapping

| Status | When |
|--------|------|
| 200 | Successful GET, PUT, PATCH |
| 201 | Successful POST (created) |
| 204 | Successful DELETE |
| 400 | Validation error, malformed request |
| 401 | Missing/invalid authentication |
| 403 | Authenticated but unauthorized |
| 404 | Resource not found |
| 409 | Conflict (duplicate, state conflict) |
| 422 | Valid syntax but unprocessable |
| 429 | Rate limited |
| 500 | Internal server error |

## Go/Fiber Specifics

**Middleware ordering:**
```go
app.Use(logger.New())
app.Use(recover.New())
app.Use(cors.New(cors.Config{
    AllowOrigins: "http://localhost:5173",
}))
app.Use(requestid.New())
// auth middleware on protected routes only
```

**Handler pattern (using generic validator from `pkg/validator.go`):**
```go
func (h *Handler) Create(c fiber.Ctx) error {
    body, err := pkg.ValidateBody[CreateUserRequest](c)
    if err != nil {
        return err
    }
    result, err := h.useCase.Execute(c.Context(), body.ToDomain())
    if err != nil {
        return mapDomainError(err)
    }
    return c.Status(201).JSON(result.ToResponse())
}

func (h *Handler) List(c fiber.Ctx) error {
    query, err := pkg.ValidateQuery[ListUsersQuery](c)
    if err != nil {
        return err
    }
    items, total, err := h.useCase.List(c.Context(), query.Limit, query.Offset)
    if err != nil {
        return mapDomainError(err)
    }
    return c.JSON(pkg.PaginatedResponse{
        Items:      items,
        Pagination: pkg.PaginationResponse{Limit: query.Limit, Offset: query.Offset, Total: total},
    })
}
```

**Generic validation functions** (see `pkg/validator.go`):
- `ValidateBody[T](c)` — parse + validate JSON body, reject unknown fields, apply defaults
- `ValidateQuery[T](c)` — parse + validate query params, handle comma-separated slices
- `ValidateParams[T](c)` — parse + validate path params
- `ValidateHeader[T](c)` — parse + validate headers

Uses `go-playground/validator/v10` with struct tags + custom validators. Supports `default` tag for zero-value defaults.

**Request struct example:**
```go
type CreateUserRequest struct {
    Email string `json:"email" validate:"required,email"`
    Name  string `json:"name" validate:"required,min=2,max=100"`
    Role  string `json:"role" validate:"omitempty,oneof=admin member viewer" default:"member"`
}

type ListUsersQuery struct {
    Limit  int    `query:"limit" validate:"omitempty,min=1,max=100" default:"20"`
    Offset int    `query:"offset" validate:"omitempty,min=0" default:"0"`
    Search string `query:"search" validate:"omitempty,max=200"`
}
```

## Python/FastAPI Specifics

**Dependency injection:**
```python
async def get_db(request: Request) -> AsyncGenerator[AsyncSession, None]:
    async with request.app.state.session_factory() as session:
        yield session

@router.post("/users", status_code=201, response_model=UserResponse)
async def create_user(
    body: CreateUserRequest,
    db: AsyncSession = Depends(get_db),
    use_case: CreateUserUseCase = Depends(get_create_user_use_case),
) -> UserResponse:
    result = await use_case.execute(body.to_domain())
    return UserResponse.from_domain(result)
```

**Error handling:**
```python
@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.code, "message": str(exc)}},
    )
```

## Pagination

**Offset-based (standard pattern):**
```
GET /api/v1/users?limit=20&offset=0
GET /api/v1/users?limit=20&offset=20
```

Response includes `pagination` object with `limit`, `offset`, `total` for client-side page calculation.

## Swagger Documentation (swaggo)

Every Go handler MUST have swaggo annotations. Generate with `swag init -g cmd/api/main.go -o docs`.

### main.go Annotations

```go
import (
    swaggo "github.com/gofiber/contrib/v3/swaggo"
    _ "project/docs" // generated swagger docs
)

// @title Project API
// @version 1.0
// @description Project API documentation
// @host localhost:8000
// @BasePath /
// @schemes http

// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Type "Bearer" followed by a space and JWT token.

func main() {
    // ...
    app.Get("/docs/*", swaggo.New(swaggo.Config{}))
}
```

### Handler Annotations

Every handler function gets a `godoc` comment block:

```go
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
// @Failure 409 {object} pkg.ErrorResponse "Duplicate email"
// @Failure 500 {object} pkg.ErrorResponse "Internal server error"
// @Router /api/v1/users [post]
func (h *Handler) CreateUser(c fiber.Ctx) error { ... }

// ListUsers godoc
// @Summary List users
// @Description Get paginated list of users
// @Tags users
// @Accept json
// @Produce json
// @Security BearerAuth
// @Param limit query int false "Limit" default(20)
// @Param offset query int false "Offset" default(0)
// @Param search query string false "Search by name or email"
// @Success 200 {object} pkg.PaginatedResponse{items=[]dto.UserResponse}
// @Failure 401 {object} pkg.ErrorResponse "Unauthorized"
// @Failure 500 {object} pkg.ErrorResponse "Internal server error"
// @Router /api/v1/users [get]
func (h *Handler) ListUsers(c fiber.Ctx) error { ... }
```

### DTO/Response Struct Tags

Add `example` tags for better Swagger UI:
```go
type UserResponse struct {
    ID    uuid.UUID `json:"id" example:"550e8400-e29b-41d4-a716-446655440000"`
    Email string    `json:"email" example:"user@example.com"`
    Name  string    `json:"name" example:"John Doe"`
    Role  string    `json:"role" example:"member"`
}
```

### Annotation Quick Reference

| Annotation | Usage |
|-----------|-------|
| `@Summary` | Short description (shown in endpoint list) |
| `@Description` | Detailed description |
| `@Tags` | Group endpoints (e.g., `users`, `system`) |
| `@Security BearerAuth` | Requires auth (omit for public endpoints) |
| `@Param name body Type true "desc"` | Request body |
| `@Param name path string true "desc"` | Path parameter |
| `@Param name query int false "desc"` | Query parameter |
| `@Success 200 {object} Type` | Success response |
| `@Failure 400 {object} pkg.ErrorResponse` | Error response |
| `@Router /path [method]` | Route path and HTTP method |

## CORS for React Dev

```
Allow-Origin: http://localhost:5173
Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Allow-Headers: Content-Type, Authorization
Allow-Credentials: true
```
