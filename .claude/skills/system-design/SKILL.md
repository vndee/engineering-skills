---
name: system-design
description: Use when designing system architecture for a new project or major feature — service boundaries, data flow, API contracts, and technical trade-offs
---

# System Design

## Overview

Design the architecture before writing code. Define service boundaries, data flow, API contracts, and infrastructure. Make trade-off decisions explicit.

**Core principle:** Architecture mistakes are 100x more expensive to fix than code mistakes. Think first, build second.

## When to Use

- Starting a new project or microservice
- Adding a major feature that changes system boundaries
- Scaling beyond current architecture
- After `product-spec`, before scaffolding

## Design Process

### 1. Requirements Extraction

From the product spec, extract:

```markdown
## Technical Requirements

### Functional
- [What the system must do — derived from user stories]

### Non-Functional
- **Throughput:** [requests/sec, messages/sec]
- **Latency:** [p50, p95, p99 targets]
- **Availability:** [uptime target, e.g., 99.9%]
- **Data volume:** [storage growth rate, retention policy]
- **Consistency:** [strong vs eventual, where each applies]
- **Security:** [auth, encryption, compliance]
```

### 2. High-Level Architecture

Define the system components and how they interact:

```markdown
## Architecture

### Components
- **API Gateway** — entry point, auth, rate limiting
- **Service A** — handles [domain area]
- **Service B** — handles [domain area]
- **Database** — PostgreSQL for [what data]
- **Cache** — Redis for [what purpose]
- **Queue** — [Redis Pub/Sub / NATS / Kafka] for [what events]

### Data Flow
[Client] → [API Gateway] → [Service] → [Database]
                                     → [Cache]
                                     → [Queue] → [Worker]
```

**For most indie/startup projects, start as a modular monolith:**
```
Single Go/Python service with clean architecture
  → Split into microservices only when you have a clear reason
  → Reasons: independent scaling, different team ownership, different deploy cadence
```

### 3. API Contract Design

Define the interface between components before implementing:

```markdown
## API Contracts

### POST /api/v1/resources
Request: { field: type }
Response: { data: Resource }
Errors: 400 (validation), 401 (auth), 409 (conflict)

### GET /api/v1/resources?limit=20&offset=0
Response: { items: Resource[], pagination: { limit, offset, total } }
```

### 4. Data Model

High-level entity relationships:

```markdown
## Data Model

### Entities
- User (id, email, name, role, created_at)
- Organization (id, name, plan, created_at)
- Membership (user_id, org_id, role)

### Relationships
- User → many Organizations (through Membership)
- Organization → many Users (through Membership)

### Indexes
- users: email (unique), org_id
- memberships: (user_id, org_id) unique, org_id
```

**REQUIRED:** Use `data-model` skill for detailed schema design.

### 5. Trade-Off Decisions

Document every significant technical decision:

```markdown
## Trade-Offs

### Decision: Monolith vs Microservices
- **Chose:** Modular monolith
- **Why:** Team of 1-3, single deploy pipeline, shared database is fine at current scale
- **Revisit when:** Service needs independent scaling or different team owns a domain

### Decision: PostgreSQL vs [alternative]
- **Chose:** PostgreSQL
- **Why:** ACID transactions, JSONB for flexible data, mature ecosystem
- **Trade-off:** Horizontal scaling harder than NoSQL (acceptable at our scale)
```

### 6. Infrastructure

```markdown
## Infrastructure

### Local Development
- Docker Compose: API + Postgres + Redis
- Hot reload: air (Go) / uvicorn --reload (Python)

### Production
- [Cloud provider]: [services used]
- Database: managed Postgres
- Cache: managed Redis
- CDN: [for frontend assets]
- CI/CD: GitHub Actions → Docker → [deployment target]
```

### 7. Risk Assessment

```markdown
## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| DB becomes bottleneck | High | Read replicas, query optimization, caching layer |
| Third-party API downtime | Medium | Circuit breaker, retry with backoff, fallback |
| Data loss | Critical | Automated backups, point-in-time recovery |
```

## Quick Reference: When to Split Services

| Signal | Action |
|--------|--------|
| Same team, same deploy cadence | Keep as monolith |
| Different scaling needs (CPU vs I/O) | Consider splitting |
| Different data ownership | Consider splitting |
| Shared database works fine | Keep as monolith |
| Cross-service transactions needed | Keep as monolith |
| Team > 5 engineers on same codebase | Consider splitting |

## Chains

- **Before:** `product-spec` for requirements
- **After:** `data-model` for detailed schema → `adr` for documenting decisions
- **Then:** Scaffold skills for implementation
