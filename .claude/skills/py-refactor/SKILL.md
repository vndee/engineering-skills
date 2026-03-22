---
name: py-refactor
description: Use when refactoring Python code, optimizing performance, enforcing type safety, or improving clean architecture in a FastAPI backend
---

# Python Refactoring & Performance

## Overview

Safe refactoring patterns, type safety enforcement, and performance optimization for Python/FastAPI backends.

**Core principle:** Never refactor without characterization tests. Never optimize without profiling. Always improve type coverage.

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

## Chains

- **REQUIRED:** Use `superpowers:systematic-debugging` for performance investigation
- Write characterization tests before any refactoring
