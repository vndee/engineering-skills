---
name: fullstack-healthcheck
description: Use when onboarding to a codebase, after pulling changes, or for periodic health assessment of Go, Python, or React projects
---

# Full-Stack Health Check

## Overview

Comprehensive project diagnostic that auto-detects stack and runs all relevant checks. Produces a health report with action items.

**Core principle:** Fast, automated diagnosis. Fix what matters, skip what doesn't.

## When to Use

- First time opening a project
- After `git pull` with many changes
- Before starting a new feature
- Periodic maintenance (weekly/monthly)

## Auto-Detection

Check for stack indicators:
- `go.mod` → Go stack
- `pyproject.toml` / `requirements.txt` → Python stack
- `package.json` + React in deps → React frontend
- `docker-compose.yml` → Docker setup
- `migrations/` → golang-migrate
- `alembic/` → Alembic migrations

## Diagnostic Checklist

### Build & Dependencies
```bash
# Go
go build ./...
go mod tidy && git diff --exit-code go.mod go.sum

# Python
uv sync --frozen
# or: pip install -e ".[dev]"

# React/Bun
bun install --frozen-lockfile
bun run build
```

### Test Suite
```bash
# Go
go test ./... -race -count=1

# Python
pytest tests/ -x --tb=short

# React
bun test --run
```

### Type Checking (Python)
```bash
mypy src/ --strict
```

### Linting
```bash
# Go
golangci-lint run

# Python
ruff check .
ruff format --check .

# React
bun run lint
```

### Migration Status
```bash
# golang-migrate: check if migrations apply cleanly
migrate -path migrations -database "$DATABASE_URL" up

# Alembic
alembic upgrade head
alembic check  # verify models match migrations
```

### Docker Health
```bash
docker compose build
docker compose up -d
docker compose ps  # check all services healthy
docker compose down
```

### Security Audit
```bash
# Go
govulncheck ./...

# Python
pip-audit

# Bun
bun audit
```

### Architecture Validation
```bash
# Go: domain imports nothing external
grep -r "fiber\|pgx\|redis" internal/domain/ && echo "VIOLATION" || echo "OK"

# Python: domain imports nothing external
grep -r "from src.infrastructure\|from sqlalchemy\|from fastapi" src/domain/ && echo "VIOLATION" || echo "OK"
```

## Health Report Format

```
## Project Health Report

### Summary
| Check | Status | Notes |
|-------|--------|-------|
| Build | PASS/FAIL | |
| Tests | PASS/FAIL | X/Y passing |
| Types | PASS/FAIL | N errors (Python) |
| Lint | PASS/FAIL | N warnings |
| Migrations | PASS/FAIL | |
| Docker | PASS/FAIL | |
| Security | PASS/FAIL | N vulnerabilities |
| Architecture | PASS/FAIL | N violations |

### Action Items
1. [CRITICAL] Fix failing tests
2. [HIGH] Update vulnerable dependency X
3. [MEDIUM] Fix type errors in Y
4. [LOW] Clean up lint warnings
```

## Chains

- **Failures trigger:** `dep-update`, `go-refactor` / `py-refactor` based on findings
- Run before `go-feature` / `py-feature` on unfamiliar codebases
