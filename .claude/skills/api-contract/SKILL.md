---
name: api-contract
description: Use when designing API contracts between frontend and backend, implementing contract testing, or managing API versioning and breaking changes
---

# API Contract Design

## Overview

Define the API contract before implementing frontend or backend. Contract-first development prevents integration surprises.

**Core principle:** The API contract is the agreement between frontend and backend. Change it deliberately, never accidentally.

## When to Use

- Starting a new feature that spans frontend and backend
- Frontend and backend developed in parallel
- Changing existing API endpoints
- Detecting or preventing breaking changes

## Contract-First Workflow

```
1. Define contract (OpenAPI spec or TypeScript types)
2. Frontend and backend agree on the contract
3. Implement in parallel:
   - Backend: implement endpoints matching contract
   - Frontend: build against contract with MSW mocks
4. Integration test: verify both sides match
```

## OpenAPI Spec (Source of Truth)

For Go projects, swaggo generates this from annotations. For contract-first, write the spec first:

```yaml
# openapi.yaml
openapi: 3.0.3
info:
  title: Project API
  version: 1.0.0
paths:
  /api/v1/users:
    get:
      summary: List users
      parameters:
        - name: limit
          in: query
          schema: { type: integer, default: 20 }
        - name: offset
          in: query
          schema: { type: integer, default: 0 }
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items: { $ref: '#/components/schemas/User' }
                  pagination:
                    $ref: '#/components/schemas/Pagination'
    post:
      summary: Create user
      requestBody:
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreateUserRequest' }
      responses:
        '201':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/User' }
        '400':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }

components:
  schemas:
    User:
      type: object
      properties:
        id: { type: string, format: uuid }
        email: { type: string, format: email }
        name: { type: string }
        created_at: { type: string, format: date-time }
    Pagination:
      type: object
      properties:
        limit: { type: integer }
        offset: { type: integer }
        total: { type: integer }
    ErrorResponse:
      type: object
      properties:
        code: { type: string }
        message: { type: string }
        details:
          type: array
          items:
            type: object
            properties:
              field: { type: string }
              message: { type: string }
```

## Shared Types (TypeScript)

Generate or manually maintain shared types between frontend and backend:

```typescript
// src/api/types.ts — mirrors the API contract exactly

export interface User {
  id: string
  email: string
  name: string
  created_at: string
}

export interface CreateUserRequest {
  email: string
  name: string
}

export interface PaginatedResponse<T> {
  items: T[]
  pagination: {
    limit: number
    offset: number
    total: number
  }
}

export interface ErrorResponse {
  code: string
  message: string
  details?: { field: string; message: string }[]
}
```

## Typed API Client

```typescript
// src/api/users.ts
import { api } from './client'
import type { User, CreateUserRequest, PaginatedResponse } from './types'

export const usersApi = {
  list: (params?: { limit?: number; offset?: number }) =>
    api.get<PaginatedResponse<User>>('/users', { params }),

  getById: (id: string) =>
    api.get<User>(`/users/${id}`),

  create: (data: CreateUserRequest) =>
    api.post<User>('/users', data),

  update: (id: string, data: Partial<CreateUserRequest>) =>
    api.patch<User>(`/users/${id}`, data),

  delete: (id: string) =>
    api.delete(`/users/${id}`),
}
```

## Contract Testing

Verify that frontend expectations match backend reality:

```typescript
// tests/contract/users.contract.test.ts
import { usersApi } from '../api/users'

describe('Users API Contract', () => {
  it('GET /users returns paginated response', async () => {
    const { data } = await usersApi.list({ limit: 10, offset: 0 })

    // Verify response shape matches contract
    expect(data).toHaveProperty('items')
    expect(data).toHaveProperty('pagination')
    expect(data.pagination).toHaveProperty('limit')
    expect(data.pagination).toHaveProperty('offset')
    expect(data.pagination).toHaveProperty('total')

    if (data.items.length > 0) {
      const user = data.items[0]
      expect(user).toHaveProperty('id')
      expect(user).toHaveProperty('email')
      expect(user).toHaveProperty('name')
      expect(user).toHaveProperty('created_at')
    }
  })
})
```

## Breaking Change Detection

**Breaking changes (require version bump):**
- Removing a field from response
- Changing a field's type
- Adding a required field to request
- Changing URL path
- Changing error codes

**Non-breaking changes (safe):**
- Adding optional field to request
- Adding field to response
- Adding new endpoint
- Adding new error code

**Versioning strategy:**
```
/api/v1/users  — current stable
/api/v2/users  — new version (when breaking changes needed)
```

Keep v1 running while migrating clients to v2. Deprecate v1 with a timeline.

## MSW Mocks (Frontend Development)

```typescript
// src/mocks/handlers/users.ts
import { http, HttpResponse } from 'msw'
import type { User, PaginatedResponse, CreateUserRequest } from '../../api/types'

const mockUsers: User[] = [
  { id: '1', email: 'alice@test.com', name: 'Alice', created_at: '2024-01-01T00:00:00Z' },
]

export const userHandlers = [
  http.get('/api/v1/users', ({ request }) => {
    const url = new URL(request.url)
    const limit = Number(url.searchParams.get('limit') ?? 20)
    const offset = Number(url.searchParams.get('offset') ?? 0)
    const items = mockUsers.slice(offset, offset + limit)

    return HttpResponse.json<PaginatedResponse<User>>({
      items,
      pagination: { limit, offset, total: mockUsers.length },
    })
  }),

  http.post('/api/v1/users', async ({ request }) => {
    const body = await request.json() as CreateUserRequest
    const newUser: User = {
      id: crypto.randomUUID(),
      ...body,
      created_at: new Date().toISOString(),
    }
    return HttpResponse.json(newUser, { status: 201 })
  }),
]
```

## Chains

- **REQUIRED:** Update CLAUDE.md with API contract conventions (`claude-md`)
- **Contract defined in:** `system-design` or `api-design`
- **Backend implementation:** `go-feature` / `py-feature` with swaggo annotations
- **Frontend implementation:** `react-feature` with typed API client
- **Swagger generation:** `swag init` validates backend matches contract
