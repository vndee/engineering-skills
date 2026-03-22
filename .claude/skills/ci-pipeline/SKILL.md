---
name: ci-pipeline
description: Use when creating or updating CI/CD pipelines with GitHub Actions or GitLab CI for Go, Python, or React projects
---

# CI/CD Pipeline Patterns

## Overview

CI/CD templates for Go, Python, and React projects. Covers lint, test, type-check, build, and deploy stages.

**Core principle:** Fast feedback. Parallelize independent stages. Fail fast on cheap checks.

## GitHub Actions — Go

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.23' }
      - uses: golangci/golangci-lint-action@v6

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.23' }
      - uses: actions/cache@v4
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
      - run: go test ./... -race -count=1 -coverprofile=coverage.out
      - run: go test ./... -race -count=1 -tags=integration
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/test?sslmode=disable

  build:
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.23' }
      - run: CGO_ENABLED=0 go build -o bin/api ./cmd/api
```

## GitHub Actions — Python

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .

  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --frozen
      - run: uv run mypy src/

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - uses: actions/cache@v4
        with:
          path: ~/.cache/uv
          key: ${{ runner.os }}-uv-${{ hashFiles('**/uv.lock') }}
      - run: uv sync --frozen
      - run: uv run pytest tests/ -m "not integration" --cov=src
      - run: uv run pytest tests/ -m integration
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost:5432/test

  build:
    runs-on: ubuntu-latest
    needs: [lint, type-check, test]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: false
```

## GitHub Actions — React/Bun

```yaml
  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - run: bun run lint
      - run: bun test
      - run: bun run build
```

## Stage Ordering

```
lint ──────┐
type-check ┼──→ build ──→ deploy
test ──────┘
```

Lint, type-check, and test run in parallel. Build depends on all passing.

## Migration Safety Check

Add to test job:
```yaml
- name: Check migrations
  run: |
    # Go
    migrate -path migrations -database "$DATABASE_URL" up
    migrate -path migrations -database "$DATABASE_URL" down
    migrate -path migrations -database "$DATABASE_URL" up
    # Python
    # alembic upgrade head && alembic downgrade -1 && alembic upgrade head
```

## Coverage Thresholds

```yaml
# Go: use go tool cover
- run: |
    go test ./... -coverprofile=coverage.out
    COVERAGE=$(go tool cover -func=coverage.out | tail -1 | awk '{print $3}' | tr -d '%')
    if (( $(echo "$COVERAGE < 70" | bc -l) )); then exit 1; fi

# Python: use pytest-cov
- run: uv run pytest --cov=src --cov-fail-under=70
```

## Common Mistakes

- Not using service containers for integration tests — use GitHub Actions services
- Missing `--frozen-lockfile` / `--frozen` — non-reproducible builds
- Running lint and test sequentially — parallelize with separate jobs
- Not caching dependencies — slow builds
- Skipping type-check in Python CI — catches real bugs
