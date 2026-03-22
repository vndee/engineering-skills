---
name: react-refactor
description: Use when refactoring React components, optimizing performance, improving component architecture, or reducing bundle size
---

# React Refactoring & Performance

## Overview

Safe component refactoring, performance optimization, and bundle size reduction for React/Vite/Bun frontends.

**Core principle:** Write characterization tests before refactoring. Profile before optimizing.

## Safe Refactoring Process

1. **Write characterization tests** — capture current rendering and behavior
2. **Refactor** — change structure, not user-visible behavior
3. **Verify** — all tests pass, visual regression check
4. **Profile** — only optimize what's measurably slow

## Component Decomposition

**Signals to split:**
- Component > 200 lines
- Multiple responsibilities in one component
- Deeply nested JSX (> 4 levels)
- Props drilling through 3+ levels

**Extract pattern:**
```tsx
// Before: monolith component
function UserDashboard() {
  // 200 lines of mixed concerns
}

// After: composed components
function UserDashboard() {
  return (
    <div>
      <UserHeader user={user} />
      <UserStats stats={stats} />
      <UserActivity activities={activities} />
    </div>
  )
}
```

## Extract Custom Hook

```tsx
// Before: logic in component
function UserList() {
  const [users, setUsers] = useState([])
  const [loading, setLoading] = useState(true)
  useEffect(() => { fetchUsers().then(setUsers).finally(() => setLoading(false)) }, [])
  // ...render
}

// After: extracted hook
function useUsers() {
  return useQuery({ queryKey: ['users'], queryFn: fetchUsers })
}

function UserList() {
  const { data: users, isLoading } = useUsers()
  // ...render
}
```

## Performance Patterns

**Profile first:**
```
React DevTools → Profiler → Record → Interact → Analyze
```

**Only optimize when profiling shows a problem:**

| Issue | Solution |
|-------|----------|
| Unnecessary re-renders | `React.memo` on expensive children |
| Expensive computation | `useMemo` with correct deps |
| Callback identity | `useCallback` when passed to memoized children |
| Long lists | Virtualization (`@tanstack/react-virtual`) |
| Large bundle | Code splitting with `React.lazy` |

**Remove unnecessary optimizations:**
```tsx
// Remove useMemo if computation is cheap
const fullName = `${first} ${last}` // Don't useMemo this

// Remove useCallback if child isn't memoized
const handleClick = () => { ... } // Don't useCallback this
```

## Bundle Optimization

```bash
# Analyze bundle
bunx vite-bundle-visualizer

# Common wins:
# 1. Lazy load routes
const Users = lazy(() => import('./features/users'))

# 2. Tree shake - use named imports
import { format } from 'date-fns' // not import * as dateFns

# 3. Dynamic import for heavy libs
const { Chart } = await import('chart.js')
```

## State Management Refactoring

| Smell | Fix |
|-------|-----|
| Prop drilling 3+ levels | Context or state library |
| Context re-renders everything | Split contexts by update frequency |
| Global state for server data | React Query |
| URL state in useState | Use URL params (useSearchParams) |

## Chains

- **REQUIRED:** Write characterization tests before refactoring
- Use `superpowers:systematic-debugging` for performance investigation
