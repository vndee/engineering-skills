# Claude Code Skills for Full-Stack Engineering

36 Claude Code skills that cover the entire engineering and product lifecycle — from product spec to production deploy, from scaffolding to incident response, led by an AI engineering lead that orchestrates everything.

Built for engineers who care about performance, clean architecture, and shipping quality code fast.

## Philosophy

### The Problem

AI coding tools produce code that works but doesn't scale. They write N+1 queries, dump everything in one file, forget your conventions every session, and guess what you want instead of asking. The output looks like a junior developer's first week — functional but naive.

### The Solution: An Assembly Line for Software

This project treats software development as an **assembly line**. A raw idea enters one end; production-grade, tested, documented software exits the other. Each stage adds value, enforces standards, and catches mistakes before they propagate.

```
Idea → Spec → Architecture → Schema → Code → Test → Review → Deploy → Monitor → Maintain
```

Every stage has a dedicated skill. Every skill encodes the knowledge of a senior engineer. The result: one person with Claude Code operates at the output of a full engineering team.

### Four Non-Negotiables

These are enforced by skills at every stage — not optional, not "nice to have":

1. **Performance first** — Every line written with scalability in mind. O(n) algorithms, batch database operations, pre-allocated collections, concurrent I/O. If there's an N+1 query in the code, the pipeline failed.

2. **Clean architecture** — Domain isolation, dependency rules, layers that don't leak. Code structured so it can evolve without rewriting. No spaghetti, no "we'll refactor later."

3. **Security by default** — Parameterized queries always. Input validation at every boundary. Auth middleware on every protected route. No hardcoded secrets, no SQL injection, no XSS. Security is not a feature to add later — it's baked into every skill from scaffold to deploy.

4. **Test-driven always** — No production code without a failing test first. TDD is the process, not a suggestion. Tests prove the code works; code without tests is a liability.

### Human + Agent = Pair Partners

The human brings **product vision, context, and decisions**. The agent brings **engineering expertise, discipline, and execution**. Neither works well alone.

The agent **always checks in** — at phase transitions, at design decisions, when something unexpected surfaces. It uses interactive questions (selectable options, not walls of text) so the human stays in the loop without doing busywork.

The agent **never asks about engineering best practices** — parameterized queries, indexes, error handling, test structure, security hardening — those are encoded in skills and applied automatically. It **does ask about product decisions** — scope, features, trade-offs, priorities — because those belong to the human.

The agent **never makes the same mistake twice.** Every correction, every gotcha, every project-specific pitfall gets recorded in CLAUDE.md. Future sessions read it automatically. The agent learns from its mistakes — permanently.

### Who This Is For

- **Solo founders** who need a full engineering department in their CLI
- **Senior engineers** who want their AI to match their standards, not produce junior-level code
- **Small teams** who want consistent engineering quality across all members' AI usage
- **Anyone** tired of re-explaining their stack, patterns, and conventions every session

### What This Replaces

| Without skills | With skills |
|---------------|------------|
| Re-explain conventions every session | Encoded permanently, applied automatically |
| N+1 queries, no indexes, O(n²) loops | Performance standards enforced at every layer |
| Everything in one file, no architecture | Clean architecture from line one |
| Security bolted on as afterthought | Security baked in from first scaffold |
| Agent guesses what you want | Agent asks targeted questions with selectable options |
| Same mistake every new session | Mistakes recorded in CLAUDE.md, never repeated |
| Junior-level output that needs rewriting | Senior-level output that ships |

## What This Is

[Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/skills) are reusable instruction sets that teach Claude how to approach engineering tasks. This suite covers:

- **Go** — Fiber + pgxpool + golang-migrate + swaggo
- **Python** — FastAPI + SQLAlchemy 2.0 async + Alembic + uv + mypy strict
- **React** — Vite + Bun + Vitest + testing-library + MSW
- **Shared** — Clean architecture, TDD-first, Docker, GitHub Actions, observability, security

## Skills Overview

### Orchestration (Always Active)

| Skill | Description |
|-------|-------------|
| `eng-lead` | AI engineering lead that orchestrates all skills — understands intent, clarifies requirements interactively, routes to the right skills |
| `interactive-clarify` | Enforces interactive Q&A (AskUserQuestion tool) over plaintext questions — users pick from options instead of reading walls of text |

### Product & Planning

| Skill | Description |
|-------|-------------|
| `product-spec` | PRDs, user stories, acceptance criteria, success metrics — from idea to actionable spec |
| `system-design` | Architecture design: service boundaries, data flow, API contracts, trade-off decisions |
| `data-model` | Database schema design: entities, relationships, indexes, access patterns, soft delete strategy |
| `adr` | Architecture Decision Records — document why you chose one approach over another |

### Scaffolding

| Skill | Description |
|-------|-------------|
| `go-scaffold` | Bootstrap Go/Fiber backend with clean architecture, pgxpool, golang-migrate, swaggo, Docker Compose |
| `py-scaffold` | Bootstrap FastAPI backend with SQLAlchemy 2.0 async, Alembic, uv, mypy strict, ruff |
| `react-scaffold` | Bootstrap React/Vite/Bun frontend with feature-based structure, typed API client, MSW |

### Feature Development

| Skill | Description |
|-------|-------------|
| `go-feature` | Layer-by-layer TDD: domain → use case → Postgres repo → Fiber handler with swaggo |
| `py-feature` | Layer-by-layer TDD: domain → Protocol → use case → SQLAlchemy repo → FastAPI router |
| `react-feature` | Component TDD with testing-library, hook testing, MSW, accessibility |
| `api-design` | REST conventions, response envelopes, generic validation, pagination, swaggo annotations |
| `api-contract` | Contract-first development: OpenAPI specs, shared types, contract testing, breaking change detection |
| `db-migrate` | golang-migrate workflows, safe DDL, zero-downtime migrations |
| `py-migrate` | Alembic + SQLAlchemy 2.0 workflows, async env.py, enum handling |
| `event-driven` | Async processing, background jobs, Redis Pub/Sub, webhook handling, event design |

### Testing

| Skill | Description |
|-------|-------------|
| `go-integration-test` | testcontainers-go for real Postgres/Redis, transaction rollback isolation |
| `py-integration-test` | testcontainers-python with async SQLAlchemy, factory fixtures |

### Code Quality, Review & Debugging

| Skill | Description |
|-------|-------------|
| `code-quality` | Performance standards: N+1 prevention, batch operations, O(n) algorithms, concurrent I/O |
| `review-code` | 4-agent parallel review: performance, architecture, quality, security — severity-ranked with auto-fix |
| `debug` | Hands-on toolkit: Docker inspect, curl, psql, redis-cli, pprof, py-spy, Playwright |

### Security & Observability

| Skill | Description |
|-------|-------------|
| `security` | Auth (JWT, RBAC), input validation, SQL injection prevention, rate limiting, security headers |
| `observability` | Structured logging (slog/structlog), OpenTelemetry tracing, Prometheus metrics, Sentry |
| `analytics` | Product analytics: event taxonomy, tracking implementation, funnels, A/B testing |

### DevOps & Deployment

| Skill | Description |
|-------|-------------|
| `docker-build` | Multi-stage Dockerfiles for Go, Python (uv), React (Bun + nginx), Docker Compose |
| `ci-pipeline` | GitHub Actions: lint, type-check, test with Postgres service, build, coverage |
| `deploy` | Docker/Kubernetes deployment, environment promotion, migration strategy, rollback procedures |

### Operations & Maintenance

| Skill | Description |
|-------|-------------|
| `incident-response` | Severity classification, triage, mitigation, root cause analysis, postmortem template |
| `go-refactor` | Safe refactoring with characterization tests, pprof profiling, SQL optimization |
| `py-refactor` | Refactoring with mypy compliance, py-spy profiling, SQLAlchemy N+1 detection |
| `react-refactor` | Component decomposition, bundle analysis, React Profiler, state management cleanup |
| `dep-update` | Safe dependency updates with security audits and type-check verification |
| `fullstack-healthcheck` | Auto-detect stack, run all checks, produce health report with action items |
| `onboarding` | Generate CLAUDE.md, ARCHITECTURE.md, CONTRIBUTING.md, CODEOWNERS from codebase |
| `claude-md` | Proactively maintain CLAUDE.md — every skill that changes structure, commands, or conventions updates it automatically |

## How Skills Connect

```
Every conversation:
  eng-lead (always active) → interactive-clarify → route to skills

Idea → Product:
  product-spec → system-design → data-model → adr

Project Start:
  go-scaffold / py-scaffold + react-scaffold
    → onboarding (generate docs)
    → docker-build → ci-pipeline → deploy
    → observability → security

Feature Development:
  api-contract → api-design → db-migrate / py-migrate
    → go-feature / py-feature  ←→  go-integration-test / py-integration-test
    → react-feature (parallel)
    → event-driven (async work)
    → analytics (instrumentation)

  All skills that change the project enforce:
    → code-quality (performance standards)
    → superpowers:test-driven-development (TDD)
    → claude-md (keep CLAUDE.md current)

Pre-commit:
  → review-code (4-agent review + auto-fix)

Something breaks:
  → debug (hands-on investigation)
    → superpowers:systematic-debugging (methodology)
    → incident-response (if production)

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
> /product-spec
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

- **Different framework?** Edit `go-scaffold` to use Echo/Gin, or `py-scaffold` for Django
- **Different DB?** Edit `db-migrate` for your migration tool, update `code-quality` patterns
- **Different CI?** Edit `ci-pipeline` for your CI provider
- **Different conventions?** Edit `api-design` for your response envelope and validation

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

### Key Patterns

**Generic validation (Go):**
```go
body, err := pkg.ValidateBody[CreateUserRequest](c)   // JSON body
query, err := pkg.ValidateQuery[ListUsersQuery](c)     // Query params
params, err := pkg.ValidateParams[GetUserParams](c)    // Path params
```

**Response envelope:**
```json
{ "items": [...], "pagination": { "limit": 20, "offset": 0, "total": 100 } }
{ "code": "validation_error", "message": "...", "details": [{ "field": "...", "message": "..." }] }
```

**Clean architecture:**
```
interfaces → application → domain ← infrastructure
```

## Works Best With

Designed to complement the [superpowers](https://github.com/anthropics/claude-code-plugins) plugin:

- `superpowers:test-driven-development` — TDD workflow (chained by all feature skills)
- `superpowers:systematic-debugging` — Root cause analysis (chained by debug/refactor skills)
- `superpowers:brainstorming` — Pre-implementation design exploration
- `superpowers:finishing-a-development-branch` — Branch completion workflow

## License

MIT
