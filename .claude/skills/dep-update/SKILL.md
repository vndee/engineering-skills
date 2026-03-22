---
name: dep-update
description: Use when updating dependencies, running security audits, or managing tech debt in Go, Python, or React projects
---

# Dependency Updates & Security

## Overview

Safe dependency update workflows for Go, Python (uv), and Bun. Branch per update, test after each.

**Core principle:** One category at a time. Full test suite after each update. Never batch blindly.

## Go Update Workflow

```bash
# 1. Check for updates
go list -m -u all

# 2. Update dependencies
go get -u ./...        # all deps
go get -u package@v1.2 # specific

# 3. Clean up
go mod tidy

# 4. Security audit
govulncheck ./...

# 5. Test
go test ./... -race -count=1
```

## Python Update Workflow

```bash
# 1. Check for updates
uv pip list --outdated

# 2. Update lock file
uv lock --upgrade          # all deps
uv lock --upgrade-package pkg  # specific

# 3. Sync environment
uv sync

# 4. Type check (catches breaking API changes)
mypy src/

# 5. Lint
ruff check .

# 6. Security audit
pip-audit

# 7. Test
pytest
```

## Bun/React Update Workflow

```bash
# 1. Check for updates
bun outdated

# 2. Update
bun update              # all
bun update package      # specific

# 3. Audit
bun audit  # or npm audit

# 4. Test
bun test
bun run build  # verify build still works
```

## Safe Update Process

1. Create feature branch: `git checkout -b chore/dep-updates`
2. Update one category at a time (e.g., test deps, then framework, then DB)
3. Commit after each category with passing tests
4. Full test + type-check + lint after each commit
5. PR with summary of what changed

## Update Categories

| Category | Risk | Update frequency |
|----------|------|-----------------|
| Security patches | Critical | Immediately |
| Bug fixes (patch) | Low | Weekly |
| Minor versions | Medium | Monthly |
| Major versions | High | Quarterly (plan migration) |

## Tech Debt Identification

Look for:
- Deprecated API usage in dependency changelogs
- Outdated patterns (pre-generics Go, pre-2.0 SQLAlchemy)
- Unused dependencies (`go mod tidy`, `depcheck`)
- Pinned versions with known vulnerabilities

## Chains

- **REQUIRED:** Invoke `superpowers:finishing-a-development-branch` when done
- **REQUIRED:** Update CLAUDE.md if major dependencies changed (`claude-md`)
