---
name: product-spec
description: Use when starting a new feature, product, or project — before any architecture or code, to define what to build and why
---

# Product Specification

## Overview

Transform a vague idea into an actionable product spec with user stories, acceptance criteria, and success metrics. This is the first step before any engineering work.

**Core principle:** If you can't explain what the user does and why they care, you're not ready to code.

## When to Use

- Starting a new product or feature
- User says "build X" without detailed requirements
- Before invoking `system-design` or any scaffold skill
- When requirements are ambiguous or incomplete

## Spec Structure

### 1. Problem Statement

What problem are we solving? Who has this problem? How do they currently solve it?

```markdown
## Problem
[Who] struggles with [what] because [why].
Currently they [workaround], which causes [pain].
```

**Ask these questions before writing:**
- Who is the user? (persona, not "everyone")
- What's the trigger? (when do they need this?)
- What's the current workaround? (proves the problem exists)
- What happens if we don't build this? (priority check)

### 2. Solution Overview

One paragraph. What are we building and how does it solve the problem?

```markdown
## Solution
Build [what] that allows [who] to [do what], resulting in [outcome].
```

### 3. User Stories

```markdown
## User Stories

### Must Have (P0)
- As a [user], I want to [action] so that [benefit]
  - AC: Given [context], when [action], then [result]
  - AC: Given [context], when [edge case], then [graceful handling]

### Should Have (P1)
- As a [user], I want to [action] so that [benefit]

### Nice to Have (P2)
- As a [user], I want to [action] so that [benefit]
```

**Rules for good user stories:**
- Written from user's perspective, not developer's
- Each has measurable acceptance criteria
- P0 = launch blocker, P1 = fast follow, P2 = backlog
- If you can't write AC, the story is too vague — break it down

### 4. Scope & Non-Goals

```markdown
## Scope
What's IN this version:
- Feature A
- Feature B

What's explicitly OUT:
- Feature C (future consideration)
- Feature D (different problem)
```

**Non-goals are as important as goals.** They prevent scope creep and keep the agent focused.

### 5. Success Metrics

```markdown
## Success Metrics
- [Metric]: [Target] (e.g., "User activation: 40% of signups complete onboarding")
- [Metric]: [Target] (e.g., "API latency: p95 < 200ms")
- [Metric]: [Target] (e.g., "Error rate: < 0.1%")
```

Every feature should have at least one user-facing metric and one technical metric.

### 6. Technical Constraints

```markdown
## Constraints
- Stack: [Go/Python/React]
- Auth: [JWT/OAuth/API key]
- Data: [estimated volume, growth rate]
- Integrations: [external services]
- Compliance: [GDPR, SOC2, etc.]
```

### 7. Open Questions

```markdown
## Open Questions
- [ ] How do we handle [edge case]?
- [ ] What's the migration path for existing users?
- [ ] Do we need admin tooling for this?
```

Don't pretend you have all the answers. Surface unknowns early.

## Quick Reference: Story Quality Check

| Bad | Good |
|-----|------|
| "User can manage data" | "User can export last 30 days of transactions as CSV" |
| "System should be fast" | "Search returns results in < 200ms for 95th percentile" |
| "Handle errors gracefully" | "When payment fails, show retry button with error reason" |
| "Admin dashboard" | "Admin can view daily active users and revenue by plan tier" |
| No acceptance criteria | Given/When/Then for each story |

## Anti-Patterns

- **Solution-first thinking** — describing implementation before understanding the problem
- **Developer stories** — "As a developer, I want a REST API" is not a user story
- **Scope creep in P0** — if everything is P0, nothing is P0
- **Vague acceptance criteria** — "works correctly" is not testable
- **Missing non-goals** — leads to unbounded scope

## Chains

- **Next:** Invoke `system-design` for architecture decisions
- **Then:** Invoke `superpowers:writing-plans` to create implementation plan
- **Then:** Invoke scaffold skills (`go-scaffold`, `py-scaffold`, `react-scaffold`)
