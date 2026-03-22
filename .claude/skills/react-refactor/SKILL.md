---
name: react-refactor
description: Use when refactoring React components, cleaning up legacy frontends, optimizing performance, improving component architecture, or reducing bundle size
---

# React Refactoring & Performance

## Overview

Safe component refactoring, performance optimization, and bundle size reduction for React/Vite/Bun frontends.

**Core principle:** Write characterization tests before refactoring. Profile before optimizing.

**3-strikes rule:** If the same refactoring approach fails 3 times, stop. The problem is architectural — escalate to `system-design` for a deeper review instead of thrashing.

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

## Legacy Frontend Rescue

When inheriting a messy React codebase:

### Step 1: Characterize Before Touching

```tsx
// Characterization test: capture what renders and what happens on interaction
test('UserList renders current behavior', async () => {
  render(<UserList />)
  // Document what actually appears — even if it's wrong
  expect(screen.getByRole('table')).toBeInTheDocument()
  expect(await screen.findAllByRole('row')).toHaveLength(11) // header + 10 rows
})
```

### Step 2: Identify the Worst Offenders

1. **No error boundaries** — one component crash takes down the whole app
2. **Fetch in useEffect with no cleanup** — race conditions, memory leaks
3. **Giant monolith components (500+ lines)** — impossible to test, split first
4. **Class components with lifecycle spaghetti** — migrate to hooks incrementally
5. **Inline styles everywhere** — extract to consistent styling approach
6. **No loading/error states** — blank screens on failure
7. **Direct DOM manipulation** — `document.getElementById` in React code

### Step 3: Triage and Prioritize

```bash
# Find god components
find src -name "*.tsx" -exec wc -l {} \; | sort -rn | head -20

# Find components with no tests
for f in $(find src -name "*.tsx" ! -name "*.test.*" ! -name "*.spec.*"); do
  test_file="${f%.tsx}.test.tsx"
  [ ! -f "$test_file" ] && echo "NO TEST: $f"
done

# Find unused exports
bunx ts-prune

# Find direct DOM access
grep -rn "document\.\(getElementById\|querySelector\|createElement\)" src/
```

### Step 4: Incremental Migration

**Don't rewrite — wrap and replace:**

```tsx
// 1. Wrap legacy component to add error boundary + loading state
function SafeUserList() {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<Spinner />}>
        <LegacyUserList />
      </Suspense>
    </ErrorBoundary>
  )
}

// 2. Extract data fetching from component to hook
function useUsers() {
  return useQuery({ queryKey: ['users'], queryFn: fetchUsers })
}

// 3. Replace legacy component piece by piece
function UserList() {
  const { data: users, isLoading, error } = useUsers()
  if (isLoading) return <Spinner />
  if (error) return <ErrorMessage error={error} />
  if (!users?.length) return <EmptyState />
  return <UserTable users={users} />
}
```

### Step 5: Add Missing States

Every data-fetching component needs all three:
```tsx
// Loading → Error → Empty → Data
if (isLoading) return <Skeleton />
if (error) return <ErrorAlert message={error.message} />
if (!data?.length) return <EmptyState message="No users found" />
return <UserTable users={data} />
```

### Step 6: Remove Dead Code

```bash
bunx ts-prune          # unused exports
bunx knip              # unused files, dependencies, exports
```

**Delete it.** Git has history.

## Chains

- **REQUIRED:** Write characterization tests before refactoring — no exceptions
- **REQUIRED:** Update CLAUDE.md with discovered gotchas and conventions (`claude-md`)
- Use `superpowers:systematic-debugging` for performance investigation
- **Legacy codebases:** Run `fullstack-healthcheck` first to prioritize what to fix
