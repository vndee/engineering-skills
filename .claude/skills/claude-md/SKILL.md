---
name: claude-md
description: Use after any skill completes work that changes project structure, conventions, commands, environment variables, dependencies, or architecture — proactively updates CLAUDE.md so future sessions inherit current project state
---

# CLAUDE.md Maintenance

## Overview

CLAUDE.md is the project's memory. Every Claude session reads it first. If it's stale, every session starts wrong.

**Core principle:** CLAUDE.md must reflect the project AS IT IS RIGHT NOW — not as it was when first scaffolded. Update it proactively after every meaningful change.

**Why this matters:** Without a living CLAUDE.md, every new session starts from zero. The agent makes the same mistakes — wrong import paths, deprecated commands, patterns that were already rejected. CLAUDE.md is how the agent stops repeating itself.

## When to Update

**ALWAYS update CLAUDE.md after:**

| Trigger | What to Update |
|---------|---------------|
| New command added (Makefile, script) | `## Commands` section |
| New env var required | `## Environment Variables` section |
| New directory created | `## Key Directories` section |
| New convention established | `## Conventions` section |
| Architecture change | `## Architecture` section |
| New dependency added | `## Stack` or `## Dependencies` section |
| New migration pattern | `## Commands` (migration commands) |
| New API pattern | `## Conventions` (API patterns) |
| Auth/security added | `## Conventions` + `## Environment Variables` |
| Docker/deploy setup | `## Commands` (Docker commands) |
| CI/CD configured | Note in `## Commands` or `## CI` section |
| New feature with new patterns | `## Conventions` if pattern is reusable |
| Test patterns changed | `## Testing` or `## Commands` section |
| Bug fix revealed a gotcha | `## Gotchas` section |
| Agent made a mistake that was corrected | `## Gotchas` section |
| User corrected agent behavior | `## Gotchas` section |

**DO NOT update for:**
- Adding a single endpoint (unless it introduces a new pattern)
- Bug fixes that don't change conventions (unless the bug reveals a project-specific gotcha)
- Internal refactoring that doesn't change the public interface

## How to Update

### Step 1: Read Current CLAUDE.md

Always read the existing CLAUDE.md first. Never overwrite — merge.

```bash
# Find the project's CLAUDE.md
# Could be at project root, or in .claude/ directory
```

### Step 2: Identify What Changed

Compare what you just did against what CLAUDE.md currently says:
- New commands? → Add to Commands
- New env vars? → Add to Environment Variables
- New directories? → Add to Key Directories
- New conventions? → Add to Conventions
- Changed architecture? → Update Architecture

### Step 2b: Record Mistakes and Corrections

**This is critical.** When the agent makes a mistake and the user corrects it, or when debugging reveals a project-specific gotcha, add it to the `## Gotchas` section immediately.

```markdown
## Gotchas
- Don't import from `internal/infrastructure/` in domain layer — breaks clean architecture
- `user.Email` is unique per org, not globally — use composite key (org_id, email)
- Redis session keys expire after 24h — don't cache user objects longer than that
- The `make migrate-up` command requires Postgres running — check `docker compose ps` first
- Don't use `json.Marshal` for API responses — use Fiber's `c.JSON()` which sets Content-Type
```

**The goal:** No future session should make the same mistake twice. If an agent was corrected, that correction lives in CLAUDE.md forever.

### Step 3: Update Surgically

**Add new items** — append to the right section, maintain existing order.

**Update existing items** — if a command changed, update the description.

**Remove stale items** — if a command was removed or a convention changed, update it.

**Never rewrite the whole file** — edit only the sections that changed.

### Step 4: Keep It Concise

CLAUDE.md is read every session. Every line costs context tokens.

```markdown
# GOOD — concise, actionable
- `make test-e2e` — end-to-end tests (requires running API)

# BAD — verbose, explanatory
- `make test-e2e` — this command runs the end-to-end test suite which requires
  the API server to be running on port 8000 and a Postgres database on 5432.
  Make sure to run `make migrate` first and set the DATABASE_URL env var.
```

## CLAUDE.md Structure Reference

Maintain these sections (add as needed, don't create empty sections):

```markdown
# Project Name

## Stack
[Language + framework + database + tools — one line each]

## Commands
[Every command a developer or AI agent needs — grouped by category]

## Architecture
[2-3 lines on architecture pattern + dependency rule]

## Key Directories
[Directory → purpose mapping]

## Conventions
[Patterns that MUST be followed: validation, pagination, error handling, etc.]

## Environment Variables
[Every env var with brief description]

## Testing
[Test patterns, fixtures, how to run different test types]

## API Patterns
[Response envelope, pagination format, auth header format]

## Gotchas
[Project-specific pitfalls, mistakes already made, things that look right but are wrong]
[Example: "Don't use time.Now() in domain — inject a Clock interface"]
[Example: "The users table has a unique constraint on (org_id, email), not just email"]
[Example: "Redis keys use prefix `app:` — always include it"]
```

## Integration with Other Skills

This skill is invoked as a **final step** by other skills. The calling skill does the work, then updates CLAUDE.md with what changed.

**Skills that MUST invoke claude-md:**
- `go-scaffold` / `py-scaffold` / `react-scaffold` — create initial CLAUDE.md
- `go-feature` / `py-feature` / `react-feature` — update if new patterns introduced
- `db-migrate` / `py-migrate` — update Commands if new migration commands
- `security` — update Conventions (auth pattern) + Environment Variables (JWT_SECRET, etc.)
- `observability` — update Conventions (logging pattern) + Environment Variables
- `deploy` / `docker-build` — update Commands (Docker/deploy commands)
- `ci-pipeline` — update Commands (CI commands)
- `api-design` / `api-contract` — update API Patterns / Conventions
- `event-driven` — update Architecture (async patterns) + Environment Variables
- `dep-update` — update Stack if major dependency changed
- `adr` — update Architecture if decision changes architecture
- `onboarding` — full CLAUDE.md generation (uses this skill's structure)

## Self-Check

After updating CLAUDE.md, verify:
- [ ] No duplicate entries
- [ ] No stale commands (removed or renamed)
- [ ] No missing env vars that new code requires
- [ ] All new directories documented
- [ ] Conventions section matches actual code patterns
- [ ] File is under 100 lines (trim if needed — CLAUDE.md is loaded every session)

## Chains

- **Called by:** Every skill that changes project structure, commands, or conventions
- **Calls:** Nothing — this is a leaf skill
- **Complements:** `onboarding` (initial generation vs ongoing maintenance)
