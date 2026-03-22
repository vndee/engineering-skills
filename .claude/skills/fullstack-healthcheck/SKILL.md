---
name: fullstack-healthcheck
description: Use when onboarding to a codebase, inheriting legacy code, after pulling changes, or for periodic health assessment — detects code quality debt, missing tests, architecture violations, and dead code in Go, Python, or React projects
---

# Full-Stack Health Check

## Overview

Comprehensive project diagnostic that auto-detects stack and runs all relevant checks. Produces a health report with action items.

**Core principle:** Fast, automated diagnosis. Fix what matters, skip what doesn't.

## When to Use

- First time opening a project
- Inheriting legacy code or taking over a codebase
- After `git pull` with many changes
- Before starting a new feature
- Periodic maintenance (weekly/monthly)
- When code quality feels off but you can't pinpoint why

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

### Code Quality & Technical Debt

**Test coverage gaps:**
```bash
# Go
go test ./... -coverprofile=coverage.out
go tool cover -func=coverage.out | tail -1  # total coverage
go tool cover -func=coverage.out | awk '$3 < 50 {print}'  # files under 50%

# Python
pytest --cov=src --cov-report=term-missing tests/
# Look for files with 0% or very low coverage — these are the danger zones

# React
bun test --coverage
```

**Dead code detection:**
```bash
# Go: find unused exports
# Check for functions/types only defined but never referenced
grep -rn "^func " --include="*.go" | while read line; do
  func_name=$(echo "$line" | grep -oP 'func \K\w+')
  count=$(grep -rn "$func_name" --include="*.go" | wc -l)
  [ "$count" -le 1 ] && echo "UNUSED: $line"
done

# Python: use vulture for dead code
uvx vulture src/

# React: find unused exports
bunx ts-prune
```

**Complexity hotspots:**
```bash
# Go: find large functions (likely need splitting)
grep -c "" internal/**/*.go | awk -F: '$2 > 200 {print "LARGE FILE:", $0}'

# Python: find complex functions
uvx radon cc src/ -s -n C  # functions with cyclomatic complexity >= C

# Find god files (> 500 lines)
find . -name "*.go" -o -name "*.py" -o -name "*.tsx" | xargs wc -l | sort -rn | head -20
```

**Code smell indicators:**
- [ ] Files over 500 lines — likely god objects, need splitting
- [ ] Functions over 50 lines — doing too much, extract
- [ ] Deeply nested if/else (> 3 levels) — flatten with early returns
- [ ] TODO/FIXME/HACK comments — count and prioritize
- [ ] Commented-out code — remove or restore
- [ ] No interfaces/protocols — everything is concrete, hard to test
- [ ] No error handling — bare panics (Go) or bare except (Python)
- [ ] Hardcoded values — magic numbers, string literals that should be constants
- [ ] Missing input validation — user data flows unchecked

**Legacy code indicators:**
```bash
# Count TODO/FIXME/HACK
grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.go" --include="*.py" --include="*.tsx" | wc -l

# Find files with no test counterpart
# Go: find .go files without _test.go
for f in $(find . -name "*.go" ! -name "*_test.go" -path "*/internal/*"); do
  test_file="${f%.go}_test.go"
  [ ! -f "$test_file" ] && echo "NO TEST: $f"
done

# Find commented-out code blocks
grep -rn "^[[:space:]]*//" --include="*.go" | grep -v "godoc\|nolint\|TODO\|FIXME" | head -20
```

## Health Report Format

```
## Project Health Report

### Summary
| Check | Status | Notes |
|-------|--------|-------|
| Build | PASS/FAIL | |
| Tests | PASS/FAIL | X/Y passing, Z% coverage |
| Types | PASS/FAIL | N errors (Python) |
| Lint | PASS/FAIL | N warnings |
| Migrations | PASS/FAIL | |
| Docker | PASS/FAIL | |
| Security | PASS/FAIL | N vulnerabilities |
| Architecture | PASS/FAIL | N violations |
| Code Quality | PASS/WARN/FAIL | N god files, M untested files |
| Tech Debt | LOW/MEDIUM/HIGH | N TODOs, M dead code items |

### Technical Debt Summary
- Test coverage: X% (target: 80%+)
- Files without tests: N
- God files (500+ lines): N
- Complex functions (CC > 10): N
- TODO/FIXME count: N
- Dead code items: N
- Commented-out code blocks: N

### Action Items (Prioritized)
1. [CRITICAL] Fix failing tests
2. [CRITICAL] Security vulnerabilities in dependency X
3. [HIGH] Add tests to untested files handling user input (security surface)
4. [HIGH] Split god files: list files over 500 lines
5. [MEDIUM] Fix type errors in Y
6. [MEDIUM] Remove dead code: list unused functions
7. [LOW] Clean up TODOs and commented-out code
8. [LOW] Reduce cyclomatic complexity in Z
```

## Legacy Codebase Triage

When the healthcheck reveals a codebase in bad shape (multiple FAIL/HIGH items), recommend this cleanup order:

1. **Make it build** — fix compilation/import errors first
2. **Make it safe** — fix security issues (SQL injection, missing auth, hardcoded secrets)
3. **Make it tested** — add characterization tests to critical paths before changing anything
4. **Make it clean** — fix architecture violations, split god files, remove dead code
5. **Make it fast** — optimize performance only after the above are stable

**Never refactor untested code.** Add characterization tests first, then refactor. This is non-negotiable.

## Chains

- **Failures trigger:** `dep-update`, `go-refactor` / `py-refactor` / `react-refactor` based on findings
- **Legacy code:** Route to refactor skills with characterization-test-first approach
- **REQUIRED:** Update CLAUDE.md with health report findings and technical debt items (`claude-md`)
- Run before `go-feature` / `py-feature` on unfamiliar codebases
