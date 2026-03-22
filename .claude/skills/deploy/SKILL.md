---
name: deploy
description: Use when setting up deployment pipelines, Kubernetes manifests, environment promotion, or production infrastructure for Go, Python, or React applications
---

# Deployment & Infrastructure

## Overview

Get code from CI to production safely. Covers environment setup, deployment strategies, and rollback procedures.

**Core principle:** Every deploy must be reversible. If you can't rollback in under 2 minutes, your deploy process is broken.

## When to Use

- Setting up production deployment for the first time
- Adding staging environment
- Configuring Kubernetes manifests
- Setting up environment promotion (staging -> prod)
- After `docker-build` and `ci-pipeline` are ready

## Environment Strategy

```
local (Docker Compose) -> staging -> production
```

| Environment | Purpose | Deploy trigger |
|-------------|---------|---------------|
| Local | Development | `docker compose up` |
| Staging | Testing, QA | Push to `staging` branch |
| Production | Users | Push to `main` or manual promote |

**Environment parity:** staging should mirror production as closely as possible (same DB engine, same Redis version, same env var structure).

## Docker-Based Deployment

### Build & Push

```yaml
# .github/workflows/deploy.yml
deploy:
  runs-on: ubuntu-latest
  needs: [lint, test]
  if: github.ref == 'refs/heads/main'
  steps:
    - uses: actions/checkout@v4

    - uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ghcr.io/${{ github.repository }}:${{ github.sha }}
          ghcr.io/${{ github.repository }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

### Docker Compose (VPS/Simple Deploy)

```yaml
# docker-compose.prod.yml
services:
  api:
    image: ghcr.io/org/api:latest
    restart: unless-stopped
    environment:
      APP_ENV: production
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
    ports: ["8000:8000"]
    healthcheck:
      test: wget -q --spider http://localhost:8000/health || exit 1
      interval: 10s
      retries: 3

  web:
    image: ghcr.io/org/web:latest
    restart: unless-stopped
    ports: ["80:80", "443:443"]

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    volumes: [pgdata:/var/lib/postgresql/data]
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

volumes:
  pgdata:
```

**Deploy script:**
```bash
#!/bin/bash
set -euo pipefail

echo "Pulling latest images..."
docker compose -f docker-compose.prod.yml pull

echo "Running migrations..."
docker compose -f docker-compose.prod.yml run --rm api migrate up

echo "Deploying..."
docker compose -f docker-compose.prod.yml up -d --remove-orphans

echo "Verifying health..."
sleep 5
curl -sf http://localhost:8000/health || { echo "Health check failed!"; exit 1; }

echo "Deploy complete."
```

### Kubernetes (k8s)

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: ghcr.io/org/api:TAG
          ports:
            - containerPort: 8000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: database-url
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8000
```

## Migration Strategy

**Always run migrations before deploying new code:**

```bash
# In CI/CD pipeline
# 1. Run migration
docker run --rm api migrate up
# 2. Deploy new code
kubectl set image deployment/api api=ghcr.io/org/api:$SHA
# 3. Verify
kubectl rollout status deployment/api
```

**Zero-downtime migration rules** (from `db-migrate` / `py-migrate`):
- Add columns as nullable first
- Never rename columns in one step
- Never drop columns until old code is fully replaced

## Rollback Procedures

```bash
# Docker Compose
docker compose -f docker-compose.prod.yml pull ghcr.io/org/api:PREVIOUS_TAG
docker compose -f docker-compose.prod.yml up -d

# Kubernetes
kubectl rollout undo deployment/api

# Database (if needed)
# Go
migrate -path migrations -database "$DATABASE_URL" down 1
# Python
alembic downgrade -1
```

## Environment Variables Management

```bash
# .env.example (committed — shows structure, no secrets)
APP_ENV=development
DATABASE_URL=postgres://app:secret@localhost:5432/app
REDIS_URL=redis://localhost:6379
JWT_SECRET=change-me
SENTRY_DSN=

# Production: use platform's secret management
# GitHub Actions: Settings -> Secrets
# K8s: kubectl create secret generic api-secrets --from-env-file=.env.prod
# AWS: Parameter Store or Secrets Manager
```

## Deploy Checklist

- [ ] CI passes (lint, type-check, test)
- [ ] Docker image builds successfully
- [ ] Migrations are backward-compatible
- [ ] Health check endpoint works
- [ ] Rollback procedure tested
- [ ] Environment variables configured
- [ ] Secrets are not in code or Docker image
- [ ] Monitoring/alerting configured for the new deploy

## Chains

- **REQUIRED:** Update CLAUDE.md with deployment commands and env vars (`claude-md`)
- **Before:** `docker-build` for Dockerfiles, `ci-pipeline` for CI
- **After deploy:** `observability` for monitoring
- **If broken:** `incident-response` for handling failures
