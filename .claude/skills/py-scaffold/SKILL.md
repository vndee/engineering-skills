---
name: py-scaffold
description: Use when starting a new Python/FastAPI backend, API, or microservice with SQLAlchemy, Alembic, and clean architecture
---

# Python Project Scaffold

## Overview

Bootstrap a new FastAPI backend with clean architecture, SQLAlchemy 2.0 async, Alembic, uv, and strict typing from day one.

**Core principle:** Start fully typed. mypy strict from the first file. No `Any` unless justified.

## Directory Structure

```
project/
  src/
    domain/                  # Entities, value objects, repository protocols
    application/             # Use cases, DTOs, service protocols
    infrastructure/
      postgres/              # SQLAlchemy models, repository implementations
      redis/                 # Cache implementations
      config.py              # Settings with pydantic-settings
    interfaces/
      http/                  # FastAPI routers, dependencies, middleware
        dependencies.py
        routes.py
    main.py                  # App factory with lifespan
  alembic/
    versions/
    env.py
  tests/
    unit/
    integration/
    conftest.py
  alembic.ini
  pyproject.toml
  Makefile
  Dockerfile
  docker-compose.yml
  CLAUDE.md
```

## Bootstrap Files

### main.py (App Factory)
```python
from contextlib import asynccontextmanager
from collections.abc import AsyncGenerator
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    # Startup
    engine = create_async_engine(settings.database_url)
    app.state.session_factory = async_sessionmaker(engine, expire_on_commit=False)
    yield
    # Shutdown
    await engine.dispose()

def create_app() -> FastAPI:
    app = FastAPI(lifespan=lifespan)
    app.include_router(api_router, prefix="/api/v1")
    return app
```

### pyproject.toml
```toml
[project]
name = "project-name"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.34",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.30",
    "alembic>=1.14",
    "pydantic-settings>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "httpx>=0.28",
    "testcontainers[postgres]>=4.0",
    "mypy>=1.13",
    "ruff>=0.8",
]

[tool.mypy]
strict = true
plugins = ["pydantic.mypy"]

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM", "TCH", "RUF"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
markers = ["integration: requires external services"]
```

### Makefile
```makefile
.PHONY: run test lint type-check migrate migrate-create

run:
	uvicorn src.main:create_app --factory --reload --port 8000

test:
	pytest tests/ -m "not integration"

test-integration:
	pytest tests/ -m integration

test-all:
	pytest tests/

lint:
	ruff check . && ruff format --check .

type-check:
	mypy src/

migrate:
	alembic upgrade head

migrate-create:
	alembic revision --autogenerate -m "$(msg)"
```

### docker-compose.yml
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: pg_isready -U app
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

## Initial Health Check (TDD)

**REQUIRED:** Invoke `superpowers:test-driven-development` to write `/health` endpoint.

```python
# Test first
async def test_health(client: AsyncClient) -> None:
    resp = await client.get("/health")
    assert resp.status_code == 200
    assert resp.json() == {"status": "ok"}
```

## CLAUDE.md Template

```markdown
# Project Name

## Stack
- Python 3.12+ / FastAPI
- SQLAlchemy 2.0 async + Alembic
- PostgreSQL + Redis
- uv for dependency management

## Commands
- `make run` — start with hot reload
- `make test` — unit tests
- `make test-integration` — integration tests
- `make lint` — ruff check + format
- `make type-check` — mypy strict
- `make migrate` — apply migrations
- `make migrate-create msg="description"` — new migration

## Architecture
Clean architecture: domain → application → infrastructure → interfaces
All code fully typed. mypy strict. No untyped defs.
```

## Chains

- Pairs with `react-scaffold` for full-stack
- Use `docker-build` for production Dockerfile
- Use `ci-pipeline` for GitHub Actions
