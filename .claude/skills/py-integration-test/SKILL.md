---
name: py-integration-test
description: Use when writing integration tests for Python/FastAPI services that need real PostgreSQL, Redis, or other external dependencies via testcontainers
---

# Python Integration Testing

## Overview

Integration tests with real dependencies using testcontainers-python. Test against actual Postgres with async SQLAlchemy sessions.

**Core principle:** If it talks to a database, test it with a real database. Mock nothing at the infrastructure boundary.

## When to Use

- Testing SQLAlchemy repository implementations
- Full HTTP lifecycle tests (router → use case → repo → DB)
- Verifying Alembic migrations
- Testing Redis cache logic

## Testcontainers Setup

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from typing import AsyncGenerator

@pytest.fixture(scope="session")
def postgres_url() -> Generator[str, None, None]:
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg.get_connection_url().replace("psycopg2", "asyncpg")

@pytest.fixture(scope="session")
async def engine(postgres_url: str) -> AsyncGenerator[AsyncEngine, None]:
    engine = create_async_engine(postgres_url)
    # Run migrations
    await run_migrations(postgres_url)
    yield engine
    await engine.dispose()

@pytest.fixture
async def session(engine: AsyncEngine) -> AsyncGenerator[AsyncSession, None]:
    async with engine.connect() as conn:
        trans = await conn.begin()
        session = AsyncSession(bind=conn, expire_on_commit=False)
        yield session
        await trans.rollback()
```

## Running Alembic Migrations in Tests

```python
from alembic.config import Config
from alembic import command

async def run_migrations(url: str) -> None:
    sync_url = url.replace("asyncpg", "psycopg2")
    config = Config("alembic.ini")
    config.set_main_option("sqlalchemy.url", sync_url)
    command.upgrade(config, "head")
```

## Test Isolation (Transaction Rollback)

Each test runs in a transaction that rolls back automatically:

```python
@pytest.fixture
async def session(engine: AsyncEngine) -> AsyncGenerator[AsyncSession, None]:
    async with engine.connect() as conn:
        trans = await conn.begin()
        session = AsyncSession(bind=conn, expire_on_commit=False)
        # Nested transaction for savepoint support
        nested = await conn.begin_nested()
        yield session
        if nested.is_active:
            await nested.rollback()
        await trans.rollback()
```

## Repository Test Example

```python
@pytest.mark.integration
async def test_user_repository_create(session: AsyncSession) -> None:
    repo = SQLAlchemyUserRepository(session)
    user = User(email="test@example.com", name="Test User")

    created = await repo.create(user)
    assert created.id is not None

    found = await repo.get_by_id(created.id)
    assert found is not None
    assert found.email == "test@example.com"
```

## Full HTTP Lifecycle Test

```python
@pytest.fixture
async def client(session: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    def override_session() -> AsyncGenerator[AsyncSession, None]:
        yield session

    app.dependency_overrides[get_db] = override_session
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()

@pytest.mark.integration
async def test_create_user_endpoint(client: AsyncClient) -> None:
    resp = await client.post("/api/v1/users", json={"email": "t@t.com", "name": "T"})
    assert resp.status_code == 201
```

## Factory Fixtures

```python
# tests/factories.py
from src.domain.user import User

def make_user(**overrides: Any) -> User:
    defaults: dict[str, Any] = {
        "email": f"user-{uuid4().hex[:8]}@test.com",
        "name": "Test User",
    }
    defaults.update(overrides)
    return User(**defaults)
```

## pytest Configuration

```ini
# pytest.ini
[pytest]
markers =
    integration: marks tests requiring external services
asyncio_mode = auto
```

```bash
# Run unit tests only
pytest -m "not integration"

# Run integration tests
pytest -m integration

# Run all
pytest
```

## Common Mistakes

- Not using `scope="session"` for container fixtures — spins up new Postgres per test
- Missing `expire_on_commit=False` — stale data after flush
- Forgetting to override FastAPI dependencies in tests
- Not using `asyncio_mode = auto` — manual `@pytest.mark.asyncio` everywhere
- Using `scope="session"` for session fixture — breaks test isolation
