---
name: observability
description: Use when adding logging, metrics, tracing, or error tracking to Go, Python, or React applications — structured observability from day one
---

# Observability

## Overview

Structured logging, distributed tracing, metrics, and error tracking. Without observability, you're debugging in the dark.

**Core principle:** If it's not logged, traced, and measured, it doesn't exist in production.

## Three Pillars

### 1. Structured Logging

**Never use unstructured log messages. Always key-value pairs.**

**Go (slog — standard library):**
```go
import "log/slog"

// Setup (in main.go)
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
slog.SetDefault(logger)

// Usage
slog.Info("user created",
    "user_id", user.ID,
    "email", user.Email,
    "org_id", orgID,
)

slog.Error("failed to create user",
    "error", err,
    "email", input.Email,
)

// With request context (middleware injects request_id)
slog.InfoContext(ctx, "processing request",
    "request_id", middleware.GetRequestID(ctx),
    "method", c.Method(),
    "path", c.Path(),
)
```

**Python (structlog):**
```python
import structlog

# Setup
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)
logger = structlog.get_logger()

# Usage
logger.info("user_created", user_id=str(user.id), email=user.email)
logger.error("user_creation_failed", error=str(err), email=input.email)

# With request context (bind request_id in middleware)
logger = logger.bind(request_id=request_id)
logger.info("processing_request", method=request.method, path=request.url.path)
```

**Logging rules:**
- **DO:** Log at service boundaries (incoming request, outgoing call, response)
- **DO:** Log business events (user created, payment processed, invitation sent)
- **DO:** Log errors with full context (what were you trying to do, what input caused it)
- **DON'T:** Log sensitive data (passwords, tokens, PII)
- **DON'T:** Log inside hot loops
- **DON'T:** Use string interpolation in log messages — use structured fields

**Log levels:**

| Level | When |
|-------|------|
| `ERROR` | Operation failed, needs attention |
| `WARN` | Unexpected but handled (retry succeeded, fallback used) |
| `INFO` | Business events, request lifecycle |
| `DEBUG` | Detailed debugging info (disabled in prod) |

### 2. Distributed Tracing (OpenTelemetry)

**Go:**
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

// Create span for a unit of work
tracer := otel.Tracer("service-name")

func (uc *CreateUserUseCase) Execute(ctx context.Context, input CreateUserInput) (*User, error) {
    ctx, span := tracer.Start(ctx, "CreateUser")
    defer span.End()

    span.SetAttributes(
        attribute.String("user.email", input.Email),
    )

    user, err := uc.repo.Create(ctx, input)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }
    return user, nil
}
```

**Fiber middleware:**
```go
import fiberotel "github.com/gofiber/contrib/v3/otel"
app.Use(fiberotel.Middleware())
```

**Python (FastAPI):**
```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

tracer = trace.get_tracer("service-name")

# Auto-instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

# Manual spans for business logic
async def create_user(self, input: CreateUserInput) -> User:
    with tracer.start_as_current_span("CreateUser") as span:
        span.set_attribute("user.email", input.email)
        try:
            return await self._repo.create(input)
        except Exception as e:
            span.record_exception(e)
            raise
```

**What to trace:**
- Every incoming HTTP request (automatic with middleware)
- Database queries (auto-instrumented with OTel libraries)
- External API calls
- Message queue publish/consume
- Cache operations
- Business-critical operations

### 3. Metrics (Prometheus)

**Go:**
```go
import "github.com/prometheus/client_golang/prometheus"

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
)
```

**Key metrics to track:**

| Metric | Type | What |
|--------|------|------|
| `http_requests_total` | Counter | Request count by method, path, status |
| `http_request_duration_seconds` | Histogram | Latency distribution |
| `db_query_duration_seconds` | Histogram | Database query time |
| `db_connections_active` | Gauge | Connection pool usage |
| `business_events_total` | Counter | Domain events (signups, payments) |
| `errors_total` | Counter | Errors by type |

**Expose metrics endpoint:**
```go
// Go: /metrics endpoint
import "github.com/prometheus/client_golang/prometheus/promhttp"
app.Get("/metrics", adaptor.HTTPHandler(promhttp.Handler()))
```

## Error Tracking (Sentry)

```go
// Go
import "github.com/getsentry/sentry-go"

sentry.Init(sentry.ClientOptions{
    Dsn:              os.Getenv("SENTRY_DSN"),
    TracesSampleRate: 0.1,
    Environment:      os.Getenv("APP_ENV"),
})

// In error handler middleware
sentry.CaptureException(err)
```

```python
# Python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration

sentry_sdk.init(
    dsn=os.getenv("SENTRY_DSN"),
    traces_sample_rate=0.1,
    integrations=[FastApiIntegration()],
)
```

**React:**
```typescript
import * as Sentry from '@sentry/react'

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  integrations: [Sentry.browserTracingIntegration()],
  tracesSampleRate: 0.1,
})

// Wrap app
<Sentry.ErrorBoundary fallback={<ErrorPage />}>
  <App />
</Sentry.ErrorBoundary>
```

## Health Check Endpoint

Every service must expose a health endpoint that checks all dependencies:

```go
// GET /health
{
    "status": "ok",          // or "degraded" or "error"
    "database": "connected",
    "redis": "connected",
    "version": "1.2.3",
    "uptime": "24h30m"
}
```

## Observability Checklist for New Services

- [ ] Structured JSON logging configured (slog/structlog)
- [ ] Request ID middleware (generated + propagated)
- [ ] OpenTelemetry tracing with auto-instrumentation
- [ ] Prometheus metrics endpoint (`/metrics`)
- [ ] Sentry error tracking initialized
- [ ] Health check endpoint with dependency checks
- [ ] Log levels appropriate (INFO in prod, DEBUG in dev)
- [ ] No sensitive data in logs/traces/metrics

## Chains

- **Setup during:** `go-scaffold` or `py-scaffold`
- **Debug with:** `debug` skill uses these signals for investigation
