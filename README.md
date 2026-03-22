# Claude Code Skills for Full-Stack Engineering

A collection of 21 Claude Code skills covering the full software engineering cycle. Built for engineers who care about performance, clean architecture, and shipping quality code fast.

## What This Is

Claude Code [skills](https://docs.anthropic.com/en/docs/claude-code/skills) are reusable instruction sets that teach Claude how to approach specific engineering tasks. Instead of repeating "don't write N+1 queries" or "use testcontainers for integration tests" every session, skills encode your standards permanently.

This suite covers two backend stacks plus React, from project scaffolding to production deployment:

- **Go** — Fiber + pgxpool + golang-migrate
- **Python** — FastAPI + SQLAlchemy 2.0 async + Alembic + uv
- **React** — Vite + Bun + Vitest + testing-library
- **Shared** — Clean architecture, TDD-first, Docker, GitHub Actions

## Skills Overview

### Scaffolding

| Skill | Description |
|-------|-------------|
| `go-scaffold` | Bootstrap a Go/Fiber backend with clean architecture, pgxpool, golang-migrate, swaggo, Docker Compose, hot reload |
| `py-scaffold` | Bootstrap a FastAPI backend with SQLAlchemy 2.0 async, Alembic, uv, mypy strict, ruff |
| `react-scaffold` | Bootstrap a React/Vite/Bun frontend with feature-based structure, typed API client, MSW |

### Feature Development

| Skill | Description |
|-------|-------------|
| `go-feature` | Layer-by-layer TDD: domain entity → repository interface → use case → Postgres repo → Fiber handler with swaggo annotations |
| `py-feature` | Layer-by-layer TDD: domain entity → Protocol → use case → SQLAlchemy repo → FastAPI router, fully typed |
| `react-feature` | Component TDD with testing-library, custom hook testing, MSW for API mocking, accessibility |
| `api-design` | REST conventions for both stacks: response envelopes, generic validation, pagination, swaggo, error handling |
| `db-migrate` | golang-migrate workflows, safe DDL patterns, zero-downtime migrations, idempotency checks |
| `py-migrate` | Alembic + SQLAlchemy 2.0 workflows, async env.py, enum handling, data migrations |

### Testing

| Skill | Description |
|-------|-------------|
| `go-integration-test` | testcontainers-go for real Postgres/Redis, programmatic migrations, transaction rollback isolation |
| `py-integration-test` | testcontainers-python with async SQLAlchemy sessions, factory fixtures, FastAPI TestClient overrides |

### DevOps

| Skill | Description |
|-------|-------------|
| `docker-build` | Multi-stage Dockerfiles for Go, Python (uv), React (Bun + nginx), full-stack Docker Compose |
| `ci-pipeline` | GitHub Actions templates: lint, type-check, test (with Postgres service), build, coverage thresholds |

### Code Quality, Review & Debugging

| Skill | Description |
|-------|-------------|
| `code-quality` | Performance standards enforced across all code: N+1 prevention, batch operations, O(n) algorithms, pre-allocation, concurrent I/O |
| `review-code` | 4-agent parallel code review: performance, architecture, quality, security — severity-ranked with auto-fix |
| `debug` | Hands-on debugging toolkit: Docker inspection, API probing (curl), log analysis, database queries, Redis, browser/Playwright, pprof/py-spy |

### Maintenance

| Skill | Description |
|-------|-------------|
| `go-refactor` | Safe refactoring with characterization tests, architecture enforcement, pprof profiling, SQL optimization |
| `py-refactor` | Refactoring with mypy compliance, py-spy profiling, SQLAlchemy N+1 detection, Protocol patterns |
| `react-refactor` | Component decomposition, bundle analysis, React Profiler, state management cleanup |
| `dep-update` | Safe dependency updates for Go, Python (uv), Bun with security audits and type-check verification |
| `fullstack-healthcheck` | Auto-detect stack, run all checks (build, test, lint, types, migrations, Docker, security), produce health report |

## How Skills Connect

```
Project Start:
  go-scaffold / py-scaffold + react-scaffold
    → docker-build → ci-pipeline → fullstack-healthcheck

Feature Development:
  api-design → db-migrate / py-migrate
    → go-feature / py-feature  ←→  go-integration-test / py-integration-test
    → react-feature (parallel)

  All features enforce:
    → code-quality (performance standards)
    → superpowers:test-driven-development (TDD)

Pre-commit:
  → review-code (4-agent review + auto-fix)

Something breaks:
  → debug (hands-on investigation)
    → superpowers:systematic-debugging (methodology)

Maintenance:
  fullstack-healthcheck
    → dep-update / go-refactor / py-refactor / react-refactor
```

## Installation

### Quick Install

```bash
# Clone the repo
git clone https://github.com/vndee/engineering-skills.git
cd engineering-skills

# Symlink all skills to Claude Code's skill directory
mkdir -p ~/.claude/skills
for skill in .claude/skills/*/; do
  name=$(basename "$skill")
  ln -sf "$(pwd)/$skill" "$HOME/.claude/skills/$name"
done
```

### Verify Installation

Open Claude Code and check that skills appear:

```
> /review-code
> /go-scaffold
> /fullstack-healthcheck
```

### Update

```bash
cd engineering-skills
git pull
# Symlinks point to the repo — no re-linking needed
```

## Customization

### Adapting to Your Stack

Each skill is a standalone Markdown file in `.claude/skills/<name>/SKILL.md`. Fork and modify:

- **Different framework?** Edit `go-scaffold` to use Echo/Gin instead of Fiber, or `py-scaffold` for Django
- **Different DB?** Edit `db-migrate` for your migration tool, update query patterns in `code-quality`
- **Different CI?** Edit `ci-pipeline` for your CI provider
- **Different conventions?** Edit `api-design` for your response envelope and validation patterns

### Adding New Skills

```bash
mkdir .claude/skills/my-skill
cat > .claude/skills/my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: Use when [specific triggering conditions]
---

# My Skill

Content here...
EOF

# Symlink
ln -sf "$(pwd)/.claude/skills/my-skill" "$HOME/.claude/skills/my-skill"
```

### Key Patterns Used

**Generic validation (Go)** — Instead of manual `BodyParser` + validation per handler, we use generic functions:
```go
body, err := pkg.ValidateBody[CreateUserRequest](c)   // JSON body
query, err := pkg.ValidateQuery[ListUsersQuery](c)     // Query params
params, err := pkg.ValidateParams[GetUserParams](c)    // Path params
```
Uses `go-playground/validator/v10` with struct tags. See `api-design` skill for full details.

**Response envelope** — Standardized across all endpoints:
```json
// Success list
{ "items": [...], "pagination": { "limit": 20, "offset": 0, "total": 100 } }

// Error
{ "code": "validation_error", "message": "...", "details": [{ "field": "email", "message": "..." }] }
```

**Clean architecture** — Dependencies point inward. Domain imports nothing external:
```
interfaces → application → domain ← infrastructure
```

## Works Best With

These skills are designed to complement the [superpowers](https://github.com/anthropics/claude-code-plugins) plugin collection:

- `superpowers:test-driven-development` — TDD workflow (chained by all feature skills)
- `superpowers:systematic-debugging` — Root cause analysis (chained by refactor skills)
- `superpowers:finishing-a-development-branch` — Branch completion workflow (chained by dep-update)
- `superpowers:brainstorming` — Pre-implementation design exploration

## Project Structure

```
.claude/
  skills/
    go-scaffold/SKILL.md
    py-scaffold/SKILL.md
    react-scaffold/SKILL.md
    go-feature/SKILL.md
    py-feature/SKILL.md
    react-feature/SKILL.md
    api-design/SKILL.md
    db-migrate/SKILL.md
    py-migrate/SKILL.md
    go-integration-test/SKILL.md
    py-integration-test/SKILL.md
    docker-build/SKILL.md
    ci-pipeline/SKILL.md
    code-quality/SKILL.md
    review-code/SKILL.md
    debug/SKILL.md
    go-refactor/SKILL.md
    py-refactor/SKILL.md
    react-refactor/SKILL.md
    dep-update/SKILL.md
    fullstack-healthcheck/SKILL.md
```

## License

MIT
