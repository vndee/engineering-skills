---
name: analytics
description: Use when implementing product analytics, event tracking, user metrics, funnels, or A/B testing in Go, Python, or React applications
---

# Product Analytics

## Overview

Instrument your product to understand user behavior. Track what matters, ignore vanity metrics.

**Core principle:** If you can't measure it, you can't improve it. But tracking everything is as bad as tracking nothing — be intentional.

## When to Use

- Launching a new product or feature
- Need to understand user behavior
- Setting up conversion funnels
- Implementing A/B tests
- Defining success metrics for a product spec

## Event Taxonomy

### Naming Convention

```
[object]_[action] — past tense, snake_case

Examples:
  user_signed_up
  post_created
  payment_completed
  invitation_sent
  feature_activated
```

### Event Structure

```typescript
interface AnalyticsEvent {
  event: string           // "user_signed_up"
  user_id?: string        // authenticated user
  anonymous_id?: string   // pre-auth (cookie/device)
  timestamp: string       // ISO 8601
  properties: {
    // Event-specific data
    [key: string]: any
  }
  context: {
    page_url?: string
    referrer?: string
    utm_source?: string
    utm_medium?: string
    utm_campaign?: string
    device_type?: string  // mobile, desktop, tablet
    app_version?: string
  }
}
```

### Event Categories

| Category | Events | Why |
|----------|--------|-----|
| **Acquisition** | `page_viewed`, `signup_started`, `user_signed_up` | Where do users come from? |
| **Activation** | `onboarding_completed`, `first_action_taken`, `feature_activated` | Do users get value? |
| **Engagement** | `session_started`, `feature_used`, `content_created` | Are users active? |
| **Retention** | `user_returned` (daily/weekly), `subscription_renewed` | Do users come back? |
| **Revenue** | `payment_completed`, `plan_upgraded`, `plan_downgraded` | Do users pay? |
| **Referral** | `invitation_sent`, `invitation_accepted`, `share_clicked` | Do users invite others? |

## Implementation

### React (Frontend Tracking)

```typescript
// src/shared/analytics.ts
type EventProperties = Record<string, string | number | boolean>

class Analytics {
  private provider: AnalyticsProvider // PostHog, Mixpanel, or custom

  track(event: string, properties?: EventProperties): void {
    this.provider.track(event, {
      ...properties,
      page_url: window.location.href,
      timestamp: new Date().toISOString(),
    })
  }

  identify(userId: string, traits?: Record<string, any>): void {
    this.provider.identify(userId, traits)
  }

  page(name?: string): void {
    this.provider.page(name)
  }
}

export const analytics = new Analytics(provider)
```

**Usage in components:**
```tsx
function SignupForm() {
  const handleSubmit = async (data: SignupData) => {
    analytics.track('signup_started', { method: 'email' })
    try {
      await signup(data)
      analytics.track('user_signed_up', { method: 'email', plan: 'free' })
      analytics.identify(user.id, { email: user.email, plan: 'free' })
    } catch (err) {
      analytics.track('signup_failed', { error: err.message })
    }
  }
}
```

**Track page views:**
```tsx
// In router
useEffect(() => {
  analytics.page()
}, [location.pathname])
```

### Go (Backend Tracking)

```go
type AnalyticsService struct {
    client AnalyticsClient // PostHog, Segment, or custom
}

func (s *AnalyticsService) Track(ctx context.Context, userID uuid.UUID, event string, properties map[string]any) {
    if err := s.client.Enqueue(analytics.Track{
        UserId:     userID.String(),
        Event:      event,
        Properties: properties,
        Timestamp:  time.Now(),
    }); err != nil {
        slog.Error("analytics track failed", "event", event, "error", err)
    }
}

// Usage in use case
func (uc *CreateUserUseCase) Execute(ctx context.Context, input CreateUserInput) (*User, error) {
    user, err := uc.repo.Create(ctx, input)
    if err != nil {
        return nil, err
    }
    uc.analytics.Track(ctx, user.ID, "user_signed_up", map[string]any{
        "method": input.SignupMethod,
        "plan":   "free",
    })
    return user, nil
}
```

### Python (Backend Tracking)

```python
class AnalyticsService:
    def __init__(self, client: AnalyticsClient) -> None:
        self._client = client

    def track(self, user_id: UUID, event: str, properties: dict[str, Any] | None = None) -> None:
        try:
            self._client.track(
                user_id=str(user_id),
                event=event,
                properties=properties or {},
                timestamp=datetime.utcnow(),
            )
        except Exception as e:
            logger.error("analytics_track_failed", event=event, error=str(e))
```

## Key Metrics Framework

### For Every Product

| Metric | Definition | How to Measure |
|--------|-----------|----------------|
| **DAU/MAU** | Daily/Monthly active users | Unique users with `session_started` per day/month |
| **Activation rate** | % of signups who complete key action | `users with first_action_taken / user_signed_up` |
| **Retention (D1/D7/D30)** | % of users returning after N days | Users active on day N / users who signed up N days ago |
| **Churn rate** | % of users who stop using | Users inactive for 30 days / total active users |
| **Conversion rate** | % of users who pay | `payment_completed / user_signed_up` |
| **ARPU** | Average revenue per user | Total revenue / active users |

### Funnel Analysis

```
Page View → Signup Started → Signup Completed → Onboarding → First Value Action → Paid
  1000         200 (20%)        150 (75%)        100 (67%)     60 (60%)          15 (25%)
```

Track drop-off at each step. The biggest drop-off is your biggest opportunity.

## A/B Testing

```typescript
// Simple feature flag approach
function useFeatureFlag(flag: string): boolean {
  const user = useCurrentUser()
  // Hash user ID to get deterministic assignment
  const hash = hashCode(`${flag}:${user.id}`) % 100
  return hash < 50 // 50/50 split
}

// Usage
function PricingPage() {
  const showNewPricing = useFeatureFlag('new-pricing-v2')
  analytics.track('pricing_page_viewed', { variant: showNewPricing ? 'new' : 'control' })
  return showNewPricing ? <NewPricing /> : <OldPricing />
}
```

## Rules

- **Track actions, not page views** (page views are supplementary)
- **Track on the backend for critical events** (payment, signup — can't be blocked by ad blockers)
- **Track on the frontend for UX events** (clicks, form interactions, page navigation)
- **Never track PII in properties** (no emails, names, IPs in event properties)
- **Keep event names stable** — changing names breaks dashboards and funnels
- **Document every event** — maintain an event catalog

## Chains

- **Defined in:** `product-spec` (success metrics)
- **Instrumented during:** `go-feature` / `py-feature` / `react-feature`
