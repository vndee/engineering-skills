---
name: py-refactor
description: Use when refactoring Python code, cleaning up legacy codebases, optimizing performance, enforcing type safety, or improving clean architecture in a FastAPI backend
---

# Python Refactoring & Performance

## Overview

Safe refactoring patterns, type safety enforcement, and performance optimization for Python/FastAPI backends.

**Core principle:** Never refactor without characterization tests. Never optimize without profiling. Always improve type coverage.

**3-strikes rule:** If the same refactoring approach fails 3 times, stop. The problem is architectural — escalate to `system-design` for a deeper review instead of thrashing.

## Safe Refactoring Process

1. **Write characterization tests** — capture current behavior
2. **Run mypy** — baseline type errors
3. **Refactor** — change structure, preserve behavior
4. **Verify** — tests pass, mypy errors same or fewer
5. **Clean up** — remove temporary tests if redundant

## Architecture Enforcement

```bash
# Domain should not import infrastructure
grep -r "from src.infrastructure\|from sqlalchemy\|from fastapi" src/domain/

# Application should not import infrastructure
grep -r "from src.infrastructure\|from sqlalchemy" src/application/
```

## Common Refactoring Moves

| Smell | Move |
|-------|------|
| Fat router | Extract use case class |
| ABC with single impl | Use Protocol instead |
| Inheritance hierarchy | Composition with Protocol |
| Sync blocking calls | async with `asyncio.gather` |
| Untyped dict passing | Pydantic model or dataclass |
| God class | Split into focused services |

## Protocol Over ABC

```python
# Before: ABC (tight coupling)
from abc import ABC, abstractmethod
class UserRepository(ABC):
    @abstractmethod
    async def create(self, user: User) -> User: ...

# After: Protocol (structural typing)
from typing import Protocol
class UserRepository(Protocol):
    async def create(self, user: User) -> User: ...
```

## Async Optimization

```python
# Bad: sequential I/O
users = await user_repo.list()
posts = await post_repo.list()

# Good: concurrent I/O
users, posts = await asyncio.gather(
    user_repo.list(),
    post_repo.list(),
)
```

## SQLAlchemy Query Optimization

```python
# N+1 detection: enable echo
engine = create_async_engine(url, echo=True)

# Fix N+1 with eager loading
from sqlalchemy.orm import selectinload, joinedload

stmt = select(UserModel).options(selectinload(UserModel.posts))

# joinedload for single related object
stmt = select(PostModel).options(joinedload(PostModel.author))
```

## Performance Profiling

```bash
# py-spy for flamegraphs
py-spy record -o profile.svg --pid $(pgrep uvicorn)

# cProfile for function-level
python -m cProfile -o output.prof -m pytest tests/
```

## mypy Strict Compliance

```bash
# Find remaining type errors
mypy src/ --strict 2>&1 | head -50

# Common fixes:
# - Add return type annotations
# - Replace Any with specific types
# - Add type: ignore[specific-error] with comment explaining why
```

## Legacy Code Rescue

When working with legacy Python code — untyped, untested, messy:

### Step 1: Characterize Before Touching

```python
# Characterization test: capture current behavior as-is
def test_legacy_create_user_current_behavior():
    """Documents what legacy code ACTUALLY does — not what it should do."""
    result = legacy_create_user({"email": "test@test.com"})
    assert result["id"] is not None  # whatever it currently returns
    assert result["email"] == "test@test.com"
```

**Never refactor untested code.** Add characterization tests first.

### Step 2: Add Types Incrementally

Don't try to type the whole codebase at once. Start with boundaries:

```python
# 1. Type the public API first (routers, service interfaces)
async def create_user(body: CreateUserRequest) -> UserResponse: ...

# 2. Then type domain entities
@dataclass
class User:
    id: UUID
    email: str
    name: str

# 3. Then type infrastructure (repos, clients)
class UserRepository(Protocol):
    async def create(self, user: User) -> User: ...

# 4. Run mypy incrementally
mypy src/interfaces/ --strict     # start here
mypy src/domain/ --strict         # then here
mypy src/ --strict                # goal
```

### Step 3: Identify the Worst Offenders

Prioritize by risk:
1. **Bare `except:` or `except Exception:`** — hiding real errors
2. **SQL string concatenation** — SQL injection, fix immediately
3. **No input validation** — user data flows unchecked into DB
4. **God files (500+ lines)** — split by responsibility
5. **Sync blocking in async** — `time.sleep()`, sync DB calls in async handlers
6. **Untyped dict passing** — replace with Pydantic models or dataclasses

### Step 4: Strangler Fig Pattern

```python
# 1. Extract protocol from legacy code
class UserService(Protocol):
    async def create(self, data: CreateUserInput) -> User: ...

# 2. Legacy class implements protocol (add type annotations)
class LegacyUserService:
    async def create(self, data: CreateUserInput) -> User:
        # existing messy code stays for now
        ...

# 3. New clean implementation
class CleanUserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

    async def create(self, data: CreateUserInput) -> User:
        user = User.from_input(data)
        return await self._repo.create(user)

# 4. Swap via dependency injection
def get_user_service() -> UserService:
    return CleanUserService(repo=PostgresUserRepository(session))
```

### Step 5: Fix Dangerous Patterns

```python
# Before: bare except hides bugs
try:
    result = do_something()
except:
    pass

# After: specific exceptions, proper logging
try:
    result = do_something()
except ValueError as e:
    logger.warning("Invalid input", error=str(e))
    raise
except DatabaseError as e:
    logger.error("Database failure", error=str(e))
    raise
```

### Step 6: Remove Dead Code

```bash
# Find dead code
uvx vulture src/

# Find unused imports
ruff check --select F401 .
```

**Delete it.** Git has history.

## Chains

- **REQUIRED:** Use `superpowers:systematic-debugging` for performance investigation
- **REQUIRED:** Write characterization tests before any refactoring — no exceptions
- **REQUIRED:** Update CLAUDE.md with discovered gotchas and conventions (`claude-md`)
- **Legacy codebases:** Run `fullstack-healthcheck` first to prioritize what to fix
