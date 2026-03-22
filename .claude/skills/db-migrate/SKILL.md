---
name: db-migrate
description: Use when creating or modifying database schema in Go projects using golang-migrate with PostgreSQL
---

# Database Migrations (golang-migrate)

## Overview

Safe, reversible PostgreSQL schema changes using golang-migrate. Every migration must be idempotent and have a working rollback.

**Core principle:** Every `up.sql` must have a working `down.sql`. Verify with up → down → up cycle.

## When to Use

- Adding tables, columns, indexes, or constraints
- Modifying existing schema
- Creating enum types
- Adding seed data via migrations

## File Naming

```
migrations/
  000001_create_users_table.up.sql
  000001_create_users_table.down.sql
  000002_add_user_email_index.up.sql
  000002_add_user_email_index.down.sql
```

Sequential numbering. Descriptive names. Always pairs.

## Makefile Targets

```makefile
MIGRATE=migrate -path migrations -database "$(DATABASE_URL)"

migrate-up:
	$(MIGRATE) up

migrate-down:
	$(MIGRATE) down 1

migrate-create:
	migrate create -ext sql -dir migrations -seq $(name)

migrate-force:
	$(MIGRATE) force $(version)
```

## Safe DDL Patterns

```sql
-- Tables: always IF NOT EXISTS
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes: CONCURRENTLY to avoid locks
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users (email);

-- Columns: nullable first for zero-downtime
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone TEXT;
-- Later migration: backfill, then NOT NULL
```

## Zero-Downtime Checklist

| Operation | Safe Approach |
|-----------|--------------|
| Add column | Add as nullable → backfill → add NOT NULL constraint |
| Drop column | Stop reading → deploy → drop in next migration |
| Rename column | Add new → copy data → update code → drop old |
| Add NOT NULL | Add with DEFAULT first |
| Add index | Use `CONCURRENTLY` |
| Drop table | Remove all references first |

## PostgreSQL Type Conventions

| Go Type | Postgres Type |
|---------|--------------|
| `uuid.UUID` | `UUID` |
| `time.Time` | `TIMESTAMPTZ` |
| `string` | `TEXT` |
| `int64` | `BIGINT` |
| `float64` | `DOUBLE PRECISION` |
| `map/struct` | `JSONB` |
| `bool` | `BOOLEAN` |

## Enum Patterns

```sql
-- up.sql
CREATE TYPE user_role AS ENUM ('admin', 'member', 'viewer');
ALTER TABLE users ADD COLUMN role user_role NOT NULL DEFAULT 'member';

-- down.sql
ALTER TABLE users DROP COLUMN IF EXISTS role;
DROP TYPE IF EXISTS user_role;

-- Adding values to existing enum (cannot be in transaction)
ALTER TYPE user_role ADD VALUE IF NOT EXISTS 'moderator';
```

## Verification

Always run the idempotency check:
```bash
make migrate-up && make migrate-down && make migrate-up
```

## Common Mistakes

- Missing `down.sql` — always write rollback
- Using `VARCHAR(n)` — prefer `TEXT` in Postgres
- `TIMESTAMP` without timezone — always use `TIMESTAMPTZ`
- Non-concurrent index creation on large tables — causes locks
- Adding NOT NULL without DEFAULT — fails on existing rows

## Chains

- **REQUIRED:** Update CLAUDE.md if new migration commands are added (`claude-md`)
