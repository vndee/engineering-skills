---
name: incident-response
description: Use when production is broken, a service is down, or a critical bug is affecting users — structured incident management and postmortem
---

# Incident Response

## Overview

When production breaks, follow a structured process. Don't panic, don't guess, don't push hot fixes without understanding.

**Core principle:** Restore service first, investigate root cause second, prevent recurrence third.

## Severity Classification

| Severity | Definition | Response Time | Examples |
|----------|-----------|---------------|---------|
| **SEV1** | Service down, all users affected | Immediate | Database crashed, API 500s on all routes, auth broken |
| **SEV2** | Major feature broken, many users affected | < 30 min | Payment processing fails, search returns no results |
| **SEV3** | Minor feature broken, some users affected | < 2 hours | Export fails for large datasets, specific edge case error |
| **SEV4** | Cosmetic or minor, workaround exists | Next business day | UI alignment issue, non-critical notification delay |

## Incident Workflow

### 1. Detect & Acknowledge

```markdown
## Incident Report
- **Severity:** SEV[1-4]
- **Detected:** [timestamp]
- **Detected by:** [monitoring alert / user report / engineer]
- **Affected:** [what users/features are impacted]
- **Symptom:** [what's broken, error messages]
```

### 2. Triage (First 5 Minutes)

**REQUIRED:** Invoke `debug` skill for hands-on investigation tools.

```bash
# Quick triage checklist — run in parallel
docker compose ps                           # containers running?
curl -s http://localhost:8000/health | jq .  # API healthy?
docker compose logs --tail=50 api           # recent errors?
docker compose exec postgres pg_isready     # DB reachable?
```

**Decision: Can we restore quickly?**

| Situation | Action |
|-----------|--------|
| Bad deploy caused it | Rollback to last good version |
| Database migration broke it | `migrate down 1` or `alembic downgrade -1` |
| Config/env change | Revert config |
| Resource exhaustion | Scale up / restart |
| Unknown cause | Continue investigation |

### 3. Mitigate (Restore Service)

Priority is restoring service, NOT fixing the bug.

```bash
# Rollback deployment
git revert HEAD && git push

# Or redeploy last known good
git checkout <last-good-tag>
docker compose up -d --build

# Or restart crashed service
docker compose restart api

# Or scale up if resource issue
docker compose up -d --scale api=3
```

**Log every action taken:**
```markdown
## Timeline
- [HH:MM] Detected: API returning 500 on all routes
- [HH:MM] Triaged: Database connection pool exhausted
- [HH:MM] Mitigated: Restarted API service, connections recovered
- [HH:MM] Confirmed: Service restored, monitoring for recurrence
```

### 4. Investigate Root Cause

Only after service is restored. Use `superpowers:systematic-debugging` methodology.

```markdown
## Root Cause Analysis
- **What happened:** [technical description]
- **Why it happened:** [root cause, not just symptom]
- **Why it wasn't caught:** [gap in testing/monitoring/process]
- **Evidence:** [logs, metrics, traces that confirm the root cause]
```

### 5. Fix & Verify

```markdown
## Fix
- **PR:** [link]
- **What changed:** [description]
- **How verified:** [test that covers this case]
```

### 6. Postmortem

Write within 24 hours of resolution:

```markdown
# Postmortem: [Incident Title]

**Date:** [date]
**Duration:** [detect to resolve]
**Severity:** SEV[N]
**Author:** [name]

## Summary
[1-2 sentences: what happened and impact]

## Timeline
[Chronological list of events with timestamps]

## Root Cause
[Technical explanation of why this happened]

## Impact
- Users affected: [count or percentage]
- Duration: [how long users were impacted]
- Data loss: [yes/no, details]

## What Went Well
- [Quick detection because of monitoring]
- [Fast rollback process]

## What Went Wrong
- [No alert for this failure mode]
- [Missing integration test for this case]

## Action Items
| Action | Owner | Priority | Status |
|--------|-------|----------|--------|
| Add monitoring for [X] | [name] | P0 | TODO |
| Add integration test for [Y] | [name] | P0 | TODO |
| Improve deploy rollback docs | [name] | P1 | TODO |
```

## Runbook Template

For recurring operational tasks, create runbooks:

```markdown
# Runbook: [Service Name] — [Scenario]

## Symptoms
- [What alerts fire]
- [What users see]
- [What logs show]

## Diagnosis Steps
1. Check [X]
2. If [condition], go to step 3
3. Check [Y]

## Resolution Steps
1. [Exact command to run]
2. [Verification command]
3. [Monitoring to confirm]

## Escalation
- If unresolved after 15 min: [who to contact]
```

## Chains

- **Investigation:** Use `debug` for hands-on tools
- **Root cause:** Use `superpowers:systematic-debugging` for methodology
- **Fix:** Use `superpowers:test-driven-development` for the fix
- **Review:** Use `review-code` before deploying the fix
