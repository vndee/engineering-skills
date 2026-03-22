---
name: docker-build
description: Use when creating or updating Docker configurations, Dockerfiles, or docker-compose setups for Go, Python, or React projects
---

# Docker Build Patterns

## Overview

Production-ready Docker configurations for Go, Python/FastAPI, and React/Bun. Multi-stage builds, Docker Compose for local dev, and optimization patterns.

**Core principle:** Small images, fast builds, layer caching. Dev and prod parity.

## Go Dockerfile (Multi-stage)

```dockerfile
FROM golang:1.23-alpine AS builder
RUN apk add --no-cache git
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /bin/api ./cmd/api

FROM alpine:3.20
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /bin/api /bin/api
COPY migrations/ /migrations/
EXPOSE 8000
ENTRYPOINT ["/bin/api"]
```

## Python Dockerfile (Multi-stage with uv)

```dockerfile
FROM python:3.12-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-editable
COPY src/ src/
COPY alembic/ alembic/
COPY alembic.ini .

FROM python:3.12-slim
COPY --from=builder /app /app
WORKDIR /app
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "src.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000"]
```

## React Dockerfile (Bun + nginx)

```dockerfile
FROM oven/bun:1 AS build
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

### nginx.conf
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location /api/ {
        proxy_pass http://api:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }

    gzip on;
    gzip_types text/css application/javascript application/json;
}
```

## Docker Compose (Full Stack — Go)

```yaml
services:
  api:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgres://app:secret@postgres:5432/app?sslmode=disable
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres: { condition: service_healthy }
    volumes: ["./backend:/app"]  # dev only
    command: air  # dev only

  web:
    build: ./frontend
    ports: ["5173:5173"]
    volumes: ["./frontend/src:/app/src"]  # dev only
    command: bun dev --host  # dev only

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

## Docker Compose (Full Stack — Python)

Replace the `api` service:
```yaml
  api:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://app:secret@postgres:5432/app
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres: { condition: service_healthy }
    volumes: ["./backend:/app"]
    command: uvicorn src.main:create_app --factory --reload --host 0.0.0.0
```

## .dockerignore

```
.git
.env
node_modules
__pycache__
*.pyc
.pytest_cache
.mypy_cache
.ruff_cache
tmp/
dist/
bin/
```

## Optimization Checklist

| Technique | Benefit |
|-----------|---------|
| Multi-stage builds | Smaller final image |
| Copy dependency files first | Layer caching |
| `--frozen-lockfile` / `--frozen` | Reproducible builds |
| Alpine / slim base | Smaller image |
| `.dockerignore` | Faster context |
| `CGO_ENABLED=0` (Go) | Static binary |
| `--no-dev` (Python) | Skip dev deps |

## Health Checks

```yaml
# For API containers
healthcheck:
  test: wget -q --spider http://localhost:8000/health || exit 1
  interval: 10s
  timeout: 5s
  retries: 3
```

## Chains

- **REQUIRED:** Update CLAUDE.md with Docker commands (`claude-md`)
- **Invoked by:** `go-scaffold` / `py-scaffold` / `react-scaffold` (during project setup)
- **Feeds into:** `ci-pipeline` (build stage) → `deploy` (production deployment)
