---
name: react-feature
description: Use when implementing a new React feature, page, component, or hook in a Vite/Bun frontend with testing-library
---

# React Feature Development

## Overview

TDD workflow for React features using Vitest + testing-library. User-centric testing with MSW for API mocking.

**Core principle:** Test what users see and do, not implementation details.

## When to Use

- Adding a new page or feature module
- Building reusable components
- Creating custom hooks
- Implementing forms, data fetching, or state management

## Feature Module Structure

```
src/features/users/
  components/
    UserList.tsx
    UserList.test.tsx
    UserForm.tsx
    UserForm.test.tsx
  hooks/
    useUsers.ts
    useUsers.test.ts
  api/
    users.ts
  types/
    index.ts
  index.ts           # public API barrel export
```

## Component TDD

**Test first:**
```tsx
// UserList.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { UserList } from './UserList'

describe('UserList', () => {
  it('renders users', () => {
    render(<UserList users={[{ id: '1', name: 'Alice', email: 'a@b.com' }]} />)
    expect(screen.getByText('Alice')).toBeInTheDocument()
  })

  it('calls onDelete when delete button clicked', async () => {
    const onDelete = vi.fn()
    const user = userEvent.setup()
    render(<UserList users={[{ id: '1', name: 'Alice', email: 'a@b.com' }]} onDelete={onDelete} />)

    await user.click(screen.getByRole('button', { name: /delete/i }))
    expect(onDelete).toHaveBeenCalledWith('1')
  })
})
```

## Custom Hook Testing

```tsx
import { renderHook, waitFor } from '@testing-library/react'
import { useUsers } from './useUsers'
import { QueryClientProvider, QueryClient } from '@tanstack/react-query'

function wrapper({ children }: { children: React.ReactNode }) {
  return <QueryClientProvider client={new QueryClient()}>{children}</QueryClientProvider>
}

it('fetches users', async () => {
  const { result } = renderHook(() => useUsers(), { wrapper })
  await waitFor(() => expect(result.current.isSuccess).toBe(true))
  expect(result.current.data).toHaveLength(2)
})
```

## API Mocking with MSW

```tsx
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/v1/users', () => {
    return HttpResponse.json({
      data: [{ id: '1', name: 'Alice', email: 'a@b.com' }],
    })
  }),
]

// src/mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'
export const server = setupServer(...handlers)

// vitest.setup.ts
beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

## State Patterns

| Pattern | When |
|---------|------|
| `useState` | Local component state |
| `useReducer` | Complex state transitions |
| React Query | Server state (fetching, caching) |
| Context | Shared UI state (theme, auth) |
| URL params | Filters, pagination, selected tab |

## Loading/Error/Empty States

Always handle all three:
```tsx
if (isLoading) return <Skeleton />
if (error) return <ErrorMessage error={error} onRetry={refetch} />
if (data.length === 0) return <EmptyState message="No users found" />
return <UserList users={data} />
```

## Accessibility Checklist

- Interactive elements have accessible names (aria-label or visible text)
- Forms have associated labels
- Error messages linked with aria-describedby
- Keyboard navigation works (Tab, Enter, Escape)
- Focus management after dynamic changes

## Chains

- **REQUIRED:** Invoke `superpowers:test-driven-development`
- **REQUIRED:** Follow `code-quality` standards — virtualize long lists, debounce inputs, paginate API calls, code-split routes
- Can develop in parallel with `go-feature` or `py-feature`
