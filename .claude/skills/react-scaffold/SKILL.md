---
name: react-scaffold
description: Use when starting a new React/Vite/Bun frontend with testing-library, feature-based structure, and typed API client
---

# React Project Scaffold

## Overview

Bootstrap a React/Vite/Bun frontend with feature-based architecture, Vitest + testing-library, typed API client, and production Docker build.

**Core principle:** Feature modules with co-located tests. Type-safe API layer from day one.

## Setup

```bash
bun create vite project-name --template react-ts
cd project-name
bun add @tanstack/react-query axios
bun add -d vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event happy-dom msw
```

## Directory Structure

```
src/
  features/                  # Feature modules
    auth/
      components/
      hooks/
      api/
      types/
      index.ts
  shared/
    components/              # Shared UI components
    hooks/                   # Shared hooks
    utils/                   # Utilities
  api/
    client.ts                # Axios instance with interceptors
    types.ts                 # Shared API types (envelope, pagination)
  mocks/
    handlers.ts              # MSW handlers
    server.ts                # MSW server setup
  App.tsx
  main.tsx
  router.tsx
```

## Key Configurations

### vite.config.ts
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': 'http://localhost:8000',
    },
  },
  test: {
    globals: true,
    environment: 'happy-dom',
    setupFiles: './src/test-setup.ts',
  },
})
```

### src/test-setup.ts
```typescript
import '@testing-library/jest-dom'
import { server } from './mocks/server'

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

### Typed API Client
```typescript
// src/api/client.ts
import axios from 'axios'

export const api = axios.create({ baseURL: '/api/v1' })

export interface ApiResponse<T> {
  data: T
  meta?: { total?: number; page?: number }
}

export interface ApiError {
  error: { code: string; message: string }
}
```

## Dockerfile (Multi-stage)

```dockerfile
FROM oven/bun:1 AS build
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

## Initial Test (TDD)

**REQUIRED:** Invoke `superpowers:test-driven-development` for `App.test.tsx`.

```tsx
import { render, screen } from '@testing-library/react'
import App from './App'

it('renders app', () => {
  render(<App />)
  expect(screen.getByRole('main')).toBeInTheDocument()
})
```

## Chains

- Pairs with `go-scaffold` or `py-scaffold` for full-stack
- Use `react-feature` for adding features
