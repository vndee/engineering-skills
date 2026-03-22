---
name: py-feature
description: Use when implementing a new feature, endpoint, or use case in a Python/FastAPI backend with clean architecture, SQLAlchemy, and PostgreSQL
---

# Python Feature Development

## Overview

Layer-by-layer TDD workflow for FastAPI backends following clean architecture. Build inside-out: domain → application → infrastructure → interfaces. All code fully typed with mypy strict.

**Core principle:** Every layer gets its own test before implementation. Type safety is non-negotiable.

## Layer-by-Layer Workflow

### 1. Domain Entity + Validation

```python
# src/domain/user.py
from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID, uuid4

@dataclass(frozen=True)
class User:
    email: str
    name: str
    id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=datetime.utcnow)

    def __post_init__(self) -> None:
        if not self.email:
            raise ValueError("Email is required")
```

**Test first:** Parametrized tests for validation.

```python
@pytest.mark.parametrize("email,expected_error", [
    ("valid@test.com", None),
    ("", "Email is required"),
])
def test_create_user(email: str, expected_error: str | None) -> None:
    if expected_error:
        with pytest.raises(ValueError, match=expected_error):
            User(email=email, name="Test")
    else:
        user = User(email=email, name="Test")
        assert user.email == email
```

### 2. Repository Protocol (in domain)

```python
# src/domain/repositories.py
from typing import Protocol

class UserRepository(Protocol):
    async def create(self, user: User) -> User: ...
    async def get_by_id(self, id: UUID) -> User | None: ...
    async def get_by_email(self, email: str) -> User | None: ...
```

### 3. Use Case + Mock Tests

```python
# src/application/create_user.py
class CreateUserUseCase:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

    async def execute(self, input: CreateUserInput) -> User:
        existing = await self._repo.get_by_email(input.email)
        if existing:
            raise DuplicateEmailError(input.email)
        user = User(email=input.email, name=input.name)
        return await self._repo.create(user)
```

**Test with mocks:** Use `unittest.mock.AsyncMock` for the protocol.

### 4. SQLAlchemy Repository + Integration Test

```python
# src/infrastructure/postgres/user_repository.py
class SQLAlchemyUserRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, user: User) -> User:
        model = UserModel.from_domain(user)
        self._session.add(model)
        await self._session.flush()
        return model.to_domain()
```

**REQUIRED:** Invoke `py-integration-test` for testcontainers patterns.

### 5. FastAPI Router + HTTP Test

```python
# src/interfaces/http/users.py
router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", status_code=201, response_model=UserResponse)
async def create_user(
    body: CreateUserRequest,
    use_case: CreateUserUseCase = Depends(get_create_user),
) -> UserResponse:
    result = await use_case.execute(body.to_input())
    return UserResponse.from_domain(result)
```

**Test with httpx:**
```python
async def test_create_user(client: AsyncClient) -> None:
    resp = await client.post("/api/v1/users", json={"email": "t@t.com", "name": "T"})
    assert resp.status_code == 201
    assert resp.json()["data"]["email"] == "t@t.com"
```

## Quick Reference

| Layer | Package | Tests | Dependencies |
|-------|---------|-------|-------------|
| Domain | `src/domain/` | Unit (parametrize) | None |
| Application | `src/application/` | Unit (AsyncMock) | Domain |
| Infrastructure | `src/infrastructure/` | Integration (testcontainers) | Domain, SQLAlchemy |
| Interfaces | `src/interfaces/http/` | HTTP (httpx AsyncClient) | Application |

## Type Safety Checklist

- All functions have return type annotations
- `Protocol` for repository interfaces (not ABC)
- `Mapped[type]` for SQLAlchemy models
- Pydantic `BaseModel` for request/response
- `AsyncGenerator` for session fixtures

## Chains

- **REQUIRED:** Invoke `superpowers:test-driven-development` at each layer
- **REQUIRED:** Follow `code-quality` standards — no N+1 queries (use `selectinload`/`joinedload`), batch operations, set-based lookups, asyncio.gather for independent I/O
- **Schema changes:** Invoke `py-migrate` before implementing repository
- **API conventions:** Reference `api-design` for request/response patterns
