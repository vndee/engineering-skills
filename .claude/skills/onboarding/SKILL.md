---
name: onboarding
description: Use when onboarding to a new codebase, setting up a new engineer, or generating project documentation like ARCHITECTURE.md, CONTRIBUTING.md, or CODEOWNERS
---

# Project Onboarding & Documentation

## Overview

Generate the documentation that makes a codebase self-explanatory. New engineers (and new Claude sessions) should be productive within an hour.

**Core principle:** If it takes more than 15 minutes to set up the dev environment or more than an hour to understand the architecture, the docs have failed.

## When to Use

- Setting up a new project (after scaffolding)
- Onboarding a new team member
- New Claude session on an existing codebase
- After major architectural changes
- Periodic documentation refresh

## Documents to Generate

### 1. CLAUDE.md (For AI Agents)

The most important file. Claude Code reads this automatically.

```markdown
# Project Name

## Stack
- [Language] + [Framework]
- [Database] + [Migration tool]
- [Cache]
- [Package manager]

## Commands
- `make run` — start dev server
- `make test` — unit tests
- `make test-integration` — integration tests (requires Docker)
- `make lint` — linter
- `make type-check` — type checker (Python)
- `make swagger` — regenerate API docs
- `make migrate` — apply migrations
- `make migrate-create name=description` — new migration

## Architecture
[Brief description of architecture pattern]
Dependencies point inward: interfaces → application → domain ← infrastructure

## Key Directories
- `cmd/api/` or `src/main.py` — entry point
- `internal/domain/` or `src/domain/` — business entities, repository interfaces
- `internal/application/` or `src/application/` — use cases
- `internal/infrastructure/` or `src/infrastructure/` — DB, cache implementations
- `internal/interfaces/http/` or `src/interfaces/http/` — HTTP handlers/routers
- `migrations/` or `alembic/` — database migrations
- `pkg/` — shared utilities (validator, response types)

## Conventions
- [Validation pattern]
- [Response envelope format]
- [Pagination: offset-based with limit/offset]
- [Error handling pattern]
- [Testing patterns]

## Environment Variables
- `DATABASE_URL` — PostgreSQL connection string
- `REDIS_URL` — Redis connection string
- `JWT_SECRET` — JWT signing secret
- [other env vars]
```

### 2. ARCHITECTURE.md

```markdown
# Architecture

## Overview
[System diagram — services, databases, queues, external APIs]

## Service Map
| Service | Tech | Purpose | Port |
|---------|------|---------|------|
| API | Go/Fiber | Backend API | 8000 |
| Web | React/Vite | Frontend | 5173 |
| Postgres | 16 | Primary database | 5432 |
| Redis | 7 | Cache + sessions | 6379 |

## Clean Architecture Layers

### Domain (`internal/domain/` or `src/domain/`)
Business entities, value objects, repository interfaces.
**Rule:** Zero external imports. No framework dependencies.

### Application (`internal/application/` or `src/application/`)
Use cases, DTOs, service interfaces.
**Rule:** Depends only on domain. Contains business orchestration logic.

### Infrastructure (`internal/infrastructure/` or `src/infrastructure/`)
Database repositories, cache implementations, external API clients.
**Rule:** Implements domain interfaces. Contains all framework-specific code.

### Interfaces (`internal/interfaces/` or `src/interfaces/`)
HTTP handlers, middleware, route registration.
**Rule:** Thin layer. Parse request → call use case → format response.

## Data Flow
```
Request → Middleware → Handler → Use Case → Repository → Database
                                    ↓
                               Domain Entity
                                    ↓
Response ← Handler ← DTO ← Use Case
```

## Key Decisions
See `docs/adr/` for Architecture Decision Records.
```

### 3. CONTRIBUTING.md

```markdown
# Contributing

## Setup

### Prerequisites
- [Go 1.23+ / Python 3.12+ / Bun 1.x]
- Docker + Docker Compose
- [Other tools]

### First Time Setup
```bash
git clone [repo]
cd [project]
cp .env.example .env

# Start dependencies
docker compose up -d postgres redis

# Install dependencies
[go mod download / uv sync / bun install]

# Run migrations
make migrate

# Verify
make test
make run
```

## Development Workflow

1. Create feature branch from `main`
2. Write tests first (TDD)
3. Implement feature
4. Run full check: `make lint && make test && make test-integration`
5. Create PR with description

## Code Standards

- Follow clean architecture (dependencies point inward)
- All code must be tested (TDD workflow)
- Go: `golangci-lint` must pass
- Python: `ruff check`, `ruff format --check`, `mypy --strict` must pass
- All handlers need swaggo annotations (Go)
- Use `pkg.ValidateBody[T]` for request validation (Go)
- Offset-based pagination (`limit`/`offset`)
```

### 4. CODEOWNERS

```
# Default owner
* @username

# Backend
/backend/ @backend-team
/backend/internal/domain/ @tech-lead

# Frontend
/frontend/ @frontend-team

# Infrastructure
/docker-compose*.yml @devops
/.github/ @devops
```

## Auto-Generation from Codebase

When onboarding to an existing codebase, analyze and generate docs:

```markdown
## Steps

1. Read project structure (`ls`, `find`)
2. Read entry point (main.go / main.py / App.tsx)
3. Read package manager config (go.mod / pyproject.toml / package.json)
4. Read existing CLAUDE.md, README, or docs
5. Run `fullstack-healthcheck` skill
6. Generate/update: CLAUDE.md, ARCHITECTURE.md, CONTRIBUTING.md
```

## Onboarding Checklist for New Engineers

- [ ] Clone repo and follow CONTRIBUTING.md setup
- [ ] Dev environment running (`make run` works)
- [ ] Tests pass (`make test`)
- [ ] Read ARCHITECTURE.md
- [ ] Read 2-3 recent PRs to understand code style
- [ ] Read key ADRs in `docs/adr/`
- [ ] Make a small change (fix a typo, add a test) to verify workflow

## Chains

- **Generate after:** `go-scaffold` / `py-scaffold` / `react-scaffold`
- **Update after:** Major architectural changes, `adr` decisions
- **Use with:** `fullstack-healthcheck` for current project state
