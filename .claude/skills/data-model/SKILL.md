---
name: data-model
description: Use when designing database schema, entity relationships, or index strategy before writing migrations — for PostgreSQL with Go or Python
---

# Data Model Design

## Overview

Design your database schema before writing migrations. Think about entities, relationships, access patterns, and indexes upfront.

**Core principle:** Your schema is your most permanent decision. Code is easy to refactor, schema migrations on production data are not.

## When to Use

- Starting a new project (before first migration)
- Adding a new domain area with multiple tables
- Redesigning existing schema
- Performance issues traced to schema problems

## Design Process

### 1. Identify Entities

From user stories, extract the nouns — these are your entities:

```markdown
"User creates an organization and invites members"
→ Entities: User, Organization, Membership, Invitation

"User writes posts and receives comments"
→ Entities: User, Post, Comment
```

### 2. Define Attributes

For each entity, define columns with types:

```markdown
## Users
| Column | Type | Constraints | Notes |
|--------|------|------------|-------|
| id | UUID | PK, default gen_random_uuid() | |
| email | TEXT | NOT NULL, UNIQUE | |
| name | TEXT | NOT NULL | |
| role | user_role ENUM | NOT NULL, default 'member' | admin, member, viewer |
| avatar_url | TEXT | nullable | |
| metadata | JSONB | NOT NULL, default '{}' | extensible fields |
| created_at | TIMESTAMPTZ | NOT NULL, default NOW() | |
| updated_at | TIMESTAMPTZ | NOT NULL, default NOW() | |
```

**Type conventions (PostgreSQL):**

| Use Case | Type | Not |
|----------|------|-----|
| Identifiers | `UUID` | `SERIAL`, `BIGINT` (unless performance-critical) |
| Text | `TEXT` | `VARCHAR(n)` (Postgres treats them identically) |
| Timestamps | `TIMESTAMPTZ` | `TIMESTAMP` (always include timezone) |
| Money | `BIGINT` (cents) or `NUMERIC` | `FLOAT`, `DOUBLE` |
| Booleans | `BOOLEAN` | `INT` |
| Flexible data | `JSONB` | `JSON` (JSONB is indexed, JSON is not) |
| Enums | `CREATE TYPE ... AS ENUM` | `TEXT` with CHECK |
| Arrays | `TEXT[]`, `UUID[]` | Separate table (unless truly array data) |

### 3. Define Relationships

```markdown
## Relationships

### One-to-Many
- Organization (1) → Users (N) — via org_id FK on users
- User (1) → Posts (N) — via author_id FK on posts

### Many-to-Many
- User ↔ Organization — via memberships junction table
  - memberships(user_id, org_id, role, joined_at)
  - UNIQUE(user_id, org_id)

### One-to-One
- User (1) → Profile (1) — via user_id UNIQUE FK on profiles
```

**Junction table rules:**
- Always add `created_at` (when was this relationship created?)
- Consider adding metadata (role, permissions, status)
- Composite unique index on both FKs
- Consider if soft-delete is needed (`deleted_at`)

### 4. Index Strategy

**Rule: Every WHERE, JOIN ON, and ORDER BY column needs an index if the table will grow beyond 10k rows.**

```markdown
## Indexes

### users
- email (UNIQUE) — login lookup
- org_id — list users by org
- created_at — sort/filter by date

### posts
- author_id — user's posts
- (org_id, created_at DESC) — feed query
- (org_id, status) WHERE status = 'published' — partial index

### memberships
- (user_id, org_id) UNIQUE — prevent duplicates
- org_id — list org members
```

**Index types:**
| Type | When |
|------|------|
| B-tree (default) | Equality, range, sorting |
| GIN | JSONB fields, array fields, full-text search |
| Partial (`WHERE condition`) | Queries that filter on a specific value |
| Composite `(a, b)` | Queries that filter on a + b together |

### 5. Access Pattern Analysis

Before finalizing, list the queries your app will run and verify each one is covered:

```markdown
## Access Patterns

| Query | Frequency | Indexed? |
|-------|-----------|----------|
| Get user by email | Very high (every auth) | ✅ users.email UNIQUE |
| List org members | High | ✅ memberships.org_id |
| Get user's orgs | High | ✅ memberships.user_id |
| Feed: org posts by date | High | ✅ posts(org_id, created_at DESC) |
| Search posts by title | Medium | ❌ Need GIN index on title |
| Count users by plan | Low (admin) | ❌ OK — sequential scan fine |
```

### 6. Soft Delete vs Hard Delete

```markdown
## Deletion Strategy

| Entity | Strategy | Reason |
|--------|----------|--------|
| User | Soft (deleted_at) | Legal retention, audit trail |
| Post | Soft (deleted_at) | User might want to recover |
| Session | Hard | No retention need, high volume |
| Invitation | Hard after expiry | Short-lived, no audit need |
```

**Soft delete pattern:**
```sql
-- Add to table
deleted_at TIMESTAMPTZ DEFAULT NULL

-- Partial index (only query active records)
CREATE INDEX idx_users_active ON users (email) WHERE deleted_at IS NULL;

-- All queries must include WHERE deleted_at IS NULL
```

### 7. Schema Review Checklist

- [ ] Every table has `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [ ] Every table has `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`
- [ ] Every table has `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`
- [ ] All foreign keys have explicit `ON DELETE` behavior (CASCADE, SET NULL, or RESTRICT)
- [ ] All high-frequency WHERE columns have indexes
- [ ] No `VARCHAR(n)` — use `TEXT`
- [ ] No `TIMESTAMP` — use `TIMESTAMPTZ`
- [ ] JSONB columns have GIN indexes if queried
- [ ] Enum types are used for fixed value sets
- [ ] Junction tables have composite unique indexes
- [ ] Soft-delete tables have partial indexes on active records

## Chains

- **Before:** `product-spec` for user stories, `system-design` for architecture
- **After:** `db-migrate` or `py-migrate` to implement the schema
- **Document:** `adr` for significant schema decisions
