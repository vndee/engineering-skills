---
name: adr
description: Use when making significant technical decisions that should be documented — framework choices, architecture patterns, trade-offs, and migration decisions
---

# Architecture Decision Records

## Overview

Document why you chose one approach over another. Future you (and future Claude sessions) need this context.

**Core principle:** Decisions without documented reasoning get relitigated endlessly. Write it down once.

## When to Use

- Choosing a framework, library, or tool
- Deciding on architecture patterns (monolith vs microservice, sync vs async)
- Making trade-offs (consistency vs availability, simplicity vs flexibility)
- Changing an existing decision
- Any decision someone might ask "why did we do it this way?"

## ADR Format

```markdown
# ADR-[NNN]: [Decision Title]

**Status:** [Proposed | Accepted | Deprecated | Superseded by ADR-NNN]
**Date:** [YYYY-MM-DD]
**Deciders:** [who was involved]

## Context

[What is the situation? What forces are at play? What problem are we solving?]

## Decision

[What did we decide to do?]

We will use [choice] because [primary reason].

## Alternatives Considered

### Alternative A: [name]
- **Pros:** [list]
- **Cons:** [list]
- **Why not:** [decisive reason]

### Alternative B: [name]
- **Pros:** [list]
- **Cons:** [list]
- **Why not:** [decisive reason]

## Consequences

### Positive
- [benefit 1]
- [benefit 2]

### Negative
- [trade-off 1]
- [trade-off 2]

### Risks
- [risk 1] — mitigated by [how]

## Review Triggers

Revisit this decision when:
- [condition 1, e.g., "team grows beyond 5 engineers"]
- [condition 2, e.g., "traffic exceeds 10k req/sec"]
```

## File Organization

```
docs/
  adr/
    001-use-fiber-over-echo.md
    002-clean-architecture.md
    003-offset-pagination.md
    004-generic-validation.md
    005-monolith-first.md
```

## Common ADR Topics

| Decision | Key Trade-offs |
|----------|---------------|
| Framework choice | Ecosystem, performance, learning curve, community |
| Monolith vs microservices | Complexity, deployment, team size, scaling |
| SQL vs NoSQL | Consistency, query flexibility, schema evolution |
| REST vs GraphQL | Client flexibility, caching, complexity |
| Sync vs async processing | Latency, reliability, complexity |
| Build vs buy | Cost, customization, maintenance burden |
| Offset vs cursor pagination | Simplicity vs performance at scale |

## Anti-Patterns

- **No alternatives listed** — if you didn't consider alternatives, you didn't make a decision
- **No consequences** — every decision has trade-offs, document them
- **Too vague** — "we chose the best option" explains nothing
- **Never updated** — mark old ADRs as deprecated/superseded, don't delete them

## Chains

- **After:** `system-design` decisions, `data-model` schema decisions
- **Reference:** During `go-refactor` / `py-refactor` to understand original intent
