---
name: interactive-clarify
description: Use for every interaction where clarification is needed — enforces AskUserQuestion tool over plaintext questions for all requirement gathering and decision making
---

# Interactive Clarification

## Overview

When you need to ask the user anything, use the AskUserQuestion tool — NEVER dump plaintext questions. Users give short messages and want help figuring out what they need through interactive selection, not by reading and typing answers to a wall of text.

**Core principle:** Asking is a UI interaction, not a text dump. Make it easy to answer.

## The Rule

```
NEVER output questions as plaintext and expect the user to type answers.
ALWAYS use the AskUserQuestion tool with selectable options.
```

**No exceptions.** This applies to:
- Requirement gathering
- Design decisions
- Implementation choices
- Scope clarification
- Bug triage
- Any situation where you need user input

## How to Use AskUserQuestion

### Format Rules

1. **1-4 questions per call** — ask the most important questions first
2. **2-4 options per question** — cover the common cases
3. **Short header** (max 12 chars) — category label like "Stack", "Scope", "Auth"
4. **Concise labels** (1-5 words) — the choice itself
5. **Helpful descriptions** — explain what each option means
6. **Mark a recommended option** — add "(Recommended)" to the most likely choice
7. **"Other" is automatic** — users can always type custom input, don't add it

### Good vs Bad

**BAD — plaintext question dump:**
```
I have a few questions:
1. What stack would you like? (Go, Python, or both?)
2. Do you need authentication?
3. What kind of database schema?
4. Should I set up Docker?
5. Do you want CI/CD?
6. What about monitoring?
7. Any specific API patterns?
```

**GOOD — interactive with AskUserQuestion:**
```
AskUserQuestion with:
  Q1: "Which backend stack?" [Header: "Stack"]
    - "Go/Fiber (Recommended)" — Go backend with Fiber framework
    - "Python/FastAPI" — Python backend with FastAPI
    - "Both" — Separate Go and Python services

  Q2: "What's included in the MVP?" [Header: "Scope", multiSelect: true]
    - "Auth (JWT)" — User authentication and authorization
    - "CRUD API" — Standard resource endpoints
    - "Admin panel" — Admin-facing management UI
    - "Real-time" — WebSocket or SSE for live updates
```

### When to Use multiSelect

Use `multiSelect: true` when options are NOT mutually exclusive:
- "Which features do you want?" — can pick multiple
- "Which checks should run?" — can pick multiple

Use `multiSelect: false` (default) when options ARE mutually exclusive:
- "Which stack?" — pick one
- "What's the priority?" — pick one

### When to Use Previews

Use the `preview` field when users need to compare visual/code options:
- Different UI layouts (ASCII mockups)
- Different code patterns
- Different architecture diagrams

```
AskUserQuestion with:
  Q1: "Which response format?" [Header: "Format"]
    - "Envelope" — Wraps all responses in data/error envelope
      preview: |
        // Success
        { "data": { "id": "1", "name": "Alice" } }
        // Error
        { "error": { "code": "not_found", "message": "..." } }

    - "Flat" — Direct response without envelope
      preview: |
        // Success
        { "id": "1", "name": "Alice" }
        // Error (HTTP status only)
        404 Not Found
```

### Follow-Up Rounds

After the first round of answers, you may need to ask 1-2 more questions based on responses. This is fine — ask in rounds, not all at once.

```
Round 1: "What are you building?" + "Which stack?"
  → User picks "API service" + "Go"

Round 2: "Does it need auth?" + "What database operations?"
  → User picks "JWT auth" + "CRUD + search"

Now you have enough to start. Invoke skills.
```

**Max 3 rounds of questions.** If you need more than that, you're overcomplicating it.

## Decision Points During Work

When you encounter a decision during implementation (not just at the start):

```
AskUserQuestion with:
  Q1: "The users table needs a role system. Which approach?" [Header: "Roles"]
    - "Simple enum (Recommended)" — Single role per user (admin, member, viewer)
    - "Role-based (RBAC)" — Multiple roles with permissions table
    - "Attribute-based" — Fine-grained permissions on resources
```

**Don't silently make decisions** for things that affect the user's product. Use AskUserQuestion for:
- Schema design choices
- Auth strategy
- API design decisions
- Feature scope trade-offs
- Architecture decisions

**Do silently decide** for things that are engineering best practices:
- Using parameterized queries (always)
- Adding indexes (always)
- Writing tests (always)
- Using clean architecture (always)

## Chains

- **Used by:** `eng-lead` (orchestrator calls this for all clarification)
- **Used by:** Every skill that needs user input during execution
- **Replaces:** Plaintext questions everywhere
