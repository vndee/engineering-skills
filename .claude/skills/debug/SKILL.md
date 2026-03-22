---
name: debug
description: Use when debugging any issue in Go, Python, or React applications — provides hands-on investigation tools including Docker inspection, API probing, log analysis, database queries, and browser debugging
---

# Hands-On Debugging Toolkit

## Overview

This skill is the **toolbox** — the concrete commands and techniques for investigating bugs across the full stack. Use alongside `superpowers:systematic-debugging` which provides the **methodology** (root cause → hypothesis → test → fix).

**Core principle:** Observe before guessing. Every hypothesis must be backed by evidence gathered from actual system state — logs, API responses, database rows, container health, browser console.

## When to Use

- Application returns unexpected responses
- Service is down or unresponsive
- Data inconsistency between layers
- Frontend shows wrong state or errors
- Docker container crashes or won't start
- Integration between services is broken
- Performance degradation in production

## Phase 1: Triage — Where Is the Bug?

Before diving deep, quickly narrow down which layer is broken:

```dot
digraph triage {
    "Bug reported" -> "Can you hit the API?"
    "Can you hit the API?" -> "Frontend issue" [label="API works"]
    "Can you hit the API?" -> "Is the container running?" [label="API fails"]
    "Is the container running?" -> "Container/infra issue" [label="no"]
    "Is the container running?" -> "Check API logs" [label="yes"]
    "Check API logs" -> "Application bug" [label="error in logs"]
    "Check API logs" -> "Check database" [label="no errors"]
    "Check database" -> "Data issue" [label="bad data"]
    "Check database" -> "Network/config issue" [label="data ok"]
}
```

Run these in parallel to triage fast:

```bash
# 1. Container health
docker compose ps

# 2. API reachable?
curl -s http://localhost:8000/health | jq .

# 3. Recent logs (last 50 lines)
docker compose logs --tail=50 api

# 4. Database reachable?
docker compose exec postgres pg_isready -U app
```

## Phase 2: Layer-by-Layer Investigation

### Docker & Infrastructure

**Container won't start:**
```bash
# Check exit code and logs
docker compose ps -a
docker compose logs api --tail=100

# Check if port is already in use
lsof -i :8000

# Inspect container details (env vars, mounts, network)
docker inspect <container_id> | jq '.[0].Config.Env'
docker inspect <container_id> | jq '.[0].Mounts'
docker inspect <container_id> | jq '.[0].NetworkSettings'

# Shell into running container
docker compose exec api sh

# Shell into crashed container (override entrypoint)
docker compose run --entrypoint sh api
```

**Resource issues:**
```bash
# Check memory/CPU usage
docker stats --no-stream

# Check disk space
docker system df

# Check if OOM killed
docker inspect <container_id> | jq '.[0].State.OOMKilled'
```

**Network between containers:**
```bash
# Test connectivity from one container to another
docker compose exec api ping postgres
docker compose exec api nc -zv postgres 5432

# Check Docker network
docker network ls
docker network inspect <network_name>
```

### Backend API (Go/Python)

**Probe endpoints directly:**
```bash
# Health check
curl -s http://localhost:8000/health | jq .

# Test specific endpoint with auth
curl -s -X GET http://localhost:8000/api/v1/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" | jq .

# Test POST with body
curl -s -X POST http://localhost:8000/api/v1/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","name":"Test"}' | jq .

# Check response headers and status code
curl -sv http://localhost:8000/api/v1/users 2>&1 | grep -E "< HTTP|< Content-Type|< X-"

# Time the request
curl -o /dev/null -s -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" http://localhost:8000/api/v1/users
```

**Go-specific logging/debugging:**
```bash
# Check application logs with structured output
docker compose logs api 2>&1 | grep -i "error\|panic\|fatal"

# Enable pprof (if wired up) and check goroutines
curl -s http://localhost:6060/debug/pprof/goroutine?debug=2 | head -100

# Check for goroutine leaks
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | head -5
# If goroutine count keeps growing, you have a leak

# Memory profile
go tool pprof http://localhost:6060/debug/pprof/heap
```

**Python/FastAPI-specific debugging:**
```bash
# Check uvicorn logs
docker compose logs api 2>&1 | grep -E "ERROR|WARNING|Traceback"

# Python traceback often spans multiple lines — get full context
docker compose logs api 2>&1 | grep -A 20 "Traceback"

# Check if async event loop is blocked
# Add to code temporarily:
# import asyncio; asyncio.get_event_loop().slow_callback_duration = 0.1

# Profile with py-spy (attach to running process)
py-spy top --pid $(pgrep uvicorn)
py-spy dump --pid $(pgrep uvicorn)  # stacktrace snapshot
```

### Database

**Check data state:**
```bash
# Connect to Postgres
docker compose exec postgres psql -U app -d app

# Or from host if port is mapped
psql "postgresql://app:secret@localhost:5432/app"
```

```sql
-- Check table exists and has data
SELECT COUNT(*) FROM users;

-- Inspect specific record
SELECT * FROM users WHERE email = 'test@test.com';

-- Check for orphaned records (missing foreign key data)
SELECT u.* FROM users u
LEFT JOIN organizations o ON u.org_id = o.id
WHERE o.id IS NULL;

-- Check for duplicates
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Check recent changes
SELECT * FROM users ORDER BY updated_at DESC LIMIT 10;

-- Check migration state (golang-migrate)
SELECT * FROM schema_migrations;

-- Check migration state (Alembic)
SELECT * FROM alembic_version;
```

**Query performance:**
```sql
-- Explain slow queries
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM events WHERE user_id = $1 ORDER BY created_at DESC;

-- Find missing indexes (look for Seq Scan on large tables)
-- If "Seq Scan" on table with >10k rows → add index

-- Check table size
SELECT relname, pg_size_pretty(pg_total_relation_size(oid)) AS size
FROM pg_class WHERE relkind = 'r' ORDER BY pg_total_relation_size(oid) DESC LIMIT 10;

-- Check active connections
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- Check for locks
SELECT pid, relation::regclass, mode, granted
FROM pg_locks WHERE NOT granted;

-- Kill stuck query
SELECT pg_cancel_backend(<pid>);
```

**Redis debugging:**
```bash
# Connect
docker compose exec redis redis-cli

# Check keys
KEYS *
# (use SCAN in production, never KEYS)

# Check specific key
GET session:user123
TTL session:user123
TYPE session:user123

# Monitor all commands in real-time
MONITOR
# (use briefly — generates heavy output)

# Memory usage
INFO memory
```

### Frontend (React)

**Use the browser for visual/interaction debugging.** Invoke the `browse` skill or use Playwright MCP tools.

**Browser console inspection:**
```bash
# Navigate to the page
# Use mcp__playwright__browser_navigate to open the URL

# Take screenshot to see current state
# Use mcp__playwright__browser_take_screenshot

# Check console for errors
# Use mcp__playwright__browser_console_messages

# Check network requests (API calls)
# Use mcp__playwright__browser_network_requests

# Evaluate JavaScript in browser context
# Use mcp__playwright__browser_evaluate with expression like:
#   "document.querySelectorAll('[data-testid]').length"
#   "window.__REACT_DEVTOOLS_GLOBAL_HOOK__ ? 'DevTools available' : 'No DevTools'"
#   "JSON.stringify(localStorage)"
```

**Common frontend debugging scenarios:**

1. **API call fails silently** — Check network tab for failed requests, inspect response body
2. **State not updating** — Check React Query devtools, inspect cache keys
3. **Blank page** — Check console for errors, look for uncaught exceptions
4. **Wrong data displayed** — Trace from API response → React Query cache → component props
5. **Layout broken** — Take screenshot, inspect computed styles

**Vite/Build issues:**
```bash
# Check for build errors
bun run build 2>&1

# Check bundle for unexpected large deps
bunx vite-bundle-visualizer

# Clear cache
rm -rf node_modules/.vite
bun run dev
```

### Logs Analysis Patterns

**Structured log searching:**
```bash
# Search for errors across all services
docker compose logs 2>&1 | grep -i "error\|panic\|fatal\|traceback" | tail -30

# Filter by time window (last 5 minutes)
docker compose logs --since 5m api

# Follow logs in real-time while reproducing
docker compose logs -f api

# Correlate by request ID (if using request-id middleware)
docker compose logs api 2>&1 | grep "req-id-abc123"

# Count error frequency
docker compose logs api 2>&1 | grep -c "ERROR"
```

## Phase 3: Reproduce & Isolate

Once you know which layer is broken:

1. **Write the exact curl/command that reproduces the bug**
2. **Simplify** — remove auth, headers, body fields until you find the minimal reproduction
3. **Check the boundary** — is the request reaching the handler? Is the query hitting the DB?

```bash
# Add temporary debug logging at boundaries
# Go: log.Printf("Handler called with: %+v", req)
# Python: logger.debug(f"Handler called with: {body}")
# Then reproduce and check logs
```

## Phase 4: Fix with Evidence

**Before fixing, you must have:**
- [ ] The exact command/action that reproduces the bug
- [ ] The exact error message or wrong behavior observed
- [ ] The specific line/query/component where it breaks
- [ ] Understanding of WHY it breaks (not just WHERE)

**Then follow `superpowers:systematic-debugging` Phase 4:**
1. Write a failing test that captures the bug
2. Fix the root cause (not the symptom)
3. Verify the test passes
4. Check no other tests broke

## Quick Reference: Debug Commands by Symptom

| Symptom | First Commands |
|---------|---------------|
| Service down | `docker compose ps` → `docker compose logs --tail=50 api` |
| 500 error | `curl -v endpoint` → `docker compose logs api \| grep ERROR` |
| Slow response | `curl -w "Total: %{time_total}s"` → `EXPLAIN ANALYZE` on query |
| Wrong data | `psql` → check actual DB rows → compare with API response |
| Auth failing | `curl -v -H "Authorization: Bearer $TOKEN"` → check middleware logs |
| Container OOM | `docker stats` → `docker inspect \| jq OOMKilled` |
| DB connection fail | `docker compose exec api nc -zv postgres 5432` → check env vars |
| Frontend blank | Browser screenshot → console messages → network requests |
| Migration stuck | `SELECT * FROM schema_migrations` or `alembic current` |
| Redis miss | `redis-cli GET key` → `TTL key` → check expiration logic |

## Chains

- **REQUIRED:** Follow `superpowers:systematic-debugging` methodology — this skill provides the tools, that skill provides the process
- **Performance issues:** Use `go-refactor` / `py-refactor` after finding root cause
- **Frontend issues:** Use `browse` skill for visual debugging with Playwright
