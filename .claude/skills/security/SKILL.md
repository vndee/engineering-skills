---
name: security
description: Use when implementing authentication, authorization, input validation, or security hardening in Go, Python, or React applications
---

# Security Engineering

## Overview

Build security in from the start, not bolt it on later. Covers authentication, authorization, input handling, and common vulnerability prevention.

**Core principle:** Trust nothing from outside your system boundary. Validate everything. Fail closed.

## When to Use

- Setting up auth for a new project
- Adding role-based access control
- Handling user input
- Before deploying to production (security checklist)
- After `review-code` flags security issues

## Authentication

### JWT with Bearer Token

**Go/Fiber middleware:**
```go
func AuthMiddleware() fiber.Handler {
    return func(c fiber.Ctx) error {
        authHeader := c.Get("Authorization")
        if authHeader == "" {
            return pkg.NewError("unauthorized", "Missing authorization header", 401)
        }

        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            return pkg.NewError("unauthorized", "Invalid authorization format", 401)
        }

        claims, err := validateJWT(parts[1])
        if err != nil {
            return pkg.NewError("unauthorized", "Invalid or expired token", 401)
        }

        c.Locals("user_id", claims.UserID)
        c.Locals("user_role", claims.Role)
        return c.Next()
    }
}

// Helper to extract user ID (safe — panics caught by recover middleware)
func GetUserID(c fiber.Ctx) (uuid.UUID, error) {
    userID, ok := c.Locals("user_id").(uuid.UUID)
    if !ok {
        return uuid.Nil, pkg.NewError("unauthorized", "User not authenticated", 401)
    }
    return userID, nil
}
```

**Python/FastAPI dependency:**
```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
) -> UserClaims:
    try:
        claims = validate_jwt(credentials.credentials)
        return claims
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid or expired token")
```

### API Key Authentication

```go
func APIKeyMiddleware(validKeys map[string]bool) fiber.Handler {
    return func(c fiber.Ctx) error {
        key := c.Get("X-API-Key")
        if key == "" || !validKeys[key] {
            return pkg.NewError("unauthorized", "Invalid API key", 401)
        }
        return c.Next()
    }
}
```

## Authorization (RBAC)

```go
func RequireRole(roles ...string) fiber.Handler {
    roleSet := make(map[string]struct{}, len(roles))
    for _, r := range roles {
        roleSet[r] = struct{}{}
    }
    return func(c fiber.Ctx) error {
        userRole, _ := c.Locals("user_role").(string)
        if _, ok := roleSet[userRole]; !ok {
            return pkg.NewError("forbidden", "Insufficient permissions", 403)
        }
        return c.Next()
    }
}

// Usage
admin := api.Group("/admin", RequireRole("admin"))
admin.Get("/users", adminHandler.ListUsers)
```

**Resource-level authorization:**
```go
// Always check: does this user own this resource?
func (h *Handler) GetPost(c fiber.Ctx) error {
    userID, _ := middleware.GetUserID(c)
    postID := c.Params("id")

    post, err := h.useCase.GetByID(c.Context(), postID)
    if err != nil {
        return mapDomainError(err)
    }

    // Ownership check
    if post.AuthorID != userID {
        return pkg.NewError("forbidden", "Not your post", 403)
    }

    return c.JSON(post.ToResponse())
}
```

## Input Security

### SQL Injection Prevention

```go
// NEVER: string concatenation
query := "SELECT * FROM users WHERE email = '" + email + "'"

// ALWAYS: parameterized queries
query := "SELECT * FROM users WHERE email = $1"
row := pool.QueryRow(ctx, query, email)
```

```python
# NEVER
session.execute(text(f"SELECT * FROM users WHERE email = '{email}'"))

# ALWAYS
stmt = select(UserModel).where(UserModel.email == email)
```

### XSS Prevention

**React:** JSX auto-escapes by default. Never use `dangerouslySetInnerHTML`.

```tsx
// SAFE: auto-escaped
<p>{userInput}</p>

// DANGEROUS: only if you sanitized with DOMPurify
<p dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(htmlContent) }} />
```

### Request Validation

Always validate at the system boundary (handlers). Internal code trusts validated input.

```go
// Request structs ARE the validation boundary
type CreateUserRequest struct {
    Email string `json:"email" validate:"required,email,max=255"`
    Name  string `json:"name" validate:"required,min=2,max=100"`
}
// pkg.ValidateBody[CreateUserRequest](c) handles validation
```

### File Upload Safety

```go
// Validate file type by content, not extension
file, _ := c.FormFile("avatar")
f, _ := file.Open()
buffer := make([]byte, 512)
f.Read(buffer)
contentType := http.DetectContentType(buffer)
allowedTypes := map[string]bool{
    "image/jpeg": true, "image/png": true, "image/webp": true,
}
if !allowedTypes[contentType] {
    return pkg.NewError("bad_request", "Invalid file type", 400)
}
```

### Rate Limiting

```go
import "github.com/gofiber/fiber/v3/middleware/limiter"

// Global rate limit
app.Use(limiter.New(limiter.Config{
    Max:        100,
    Expiration: 1 * time.Minute,
}))

// Stricter limit for auth endpoints
auth := api.Group("/auth")
auth.Use(limiter.New(limiter.Config{
    Max:        10,
    Expiration: 1 * time.Minute,
}))
```

## Secret Management

```bash
# NEVER commit secrets
echo ".env" >> .gitignore

# Use environment variables
DATABASE_URL=postgres://...
JWT_SECRET=...
SENTRY_DSN=...
API_KEY=...
```

```go
// Load from environment, fail if missing
func MustGetEnv(key string) string {
    val := os.Getenv(key)
    if val == "" {
        log.Fatalf("required env var %s is not set", key)
    }
    return val
}
```

## Security Headers

```go
app.Use(func(c fiber.Ctx) error {
    c.Set("X-Content-Type-Options", "nosniff")
    c.Set("X-Frame-Options", "DENY")
    c.Set("X-XSS-Protection", "1; mode=block")
    c.Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
    c.Set("Referrer-Policy", "strict-origin-when-cross-origin")
    return c.Next()
})
```

## Pre-Deploy Security Checklist

- [ ] All endpoints have authentication (except public routes)
- [ ] Resource-level authorization checks (ownership)
- [ ] All user input validated (ValidateBody/Query/Params or Pydantic)
- [ ] Parameterized SQL queries only (no string concatenation)
- [ ] Rate limiting on auth and public endpoints
- [ ] CORS configured for specific origins (not `*`)
- [ ] Security headers set
- [ ] Secrets in environment variables (not in code)
- [ ] `.env` in `.gitignore`
- [ ] No sensitive data in logs
- [ ] File uploads validated by content type
- [ ] JWT tokens expire (not indefinite)
- [ ] HTTPS enforced in production

## Chains

- **Build into:** `go-scaffold` / `py-scaffold` (auth middleware from day one)
- **Checked by:** `review-code` Agent 4 (security review)
- **Reference:** `api-design` for auth annotations in swaggo
