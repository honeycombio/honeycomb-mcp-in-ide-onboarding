# SLO Basics Guide

This guide explains Service Level Objectives (SLOs) from Honeycomb's perspective and how to work with them effectively.

---

## What Are SLOs?

An **SLO (Service Level Objective)** is a target for how reliable your service should be. It answers the question: "How good is good enough?"

### The SLO Stack

| Term | Definition | Example |
|------|------------|---------|
| **SLI** | Service Level Indicator — the metric you measure | "Successful checkout requests" |
| **SLO** | Service Level Objective — the target for that metric | "99.9% of checkouts succeed" |
| **SLA** | Service Level Agreement — contractual commitment | "We guarantee 99.5% uptime" |

**Key insight:** SLOs should be set *tighter* than SLAs. If your SLA promises 99.5%, set your SLO at 99.9% so you have warning before breaching the contract.

---

## Anatomy of an SLO in Honeycomb

When you call `get_slos`, you'll see:

```yaml
name: "checkout-success"
target: 99.9           # 99.9% of events should succeed
time_period: 30        # Measured over 30 days
sli:
  dataset: checkout-service
  filter: "name = 'POST /checkout'"
  success_criteria: "http.status_code < 500"
status: "Normal"       # Or "Triggered" if burning too fast
compliance: 99.94      # Current compliance percentage
budget_remaining: 85   # Percentage of error budget left
```

### Reading SLO Status

| Status | Meaning | Action |
|--------|---------|--------|
| **Normal** | Meeting target, budget healthy | Continue normal work |
| **Triggered** | Burning budget faster than sustainable | Investigate and remediate |
| **No Events** | No data matching the SLI filter | Check instrumentation |

---

## Understanding Error Budgets

The **error budget** is the amount of failure you can tolerate while still meeting your SLO.

### Calculating Error Budget

```
Target: 99.9% over 30 days
Total events: 1,000,000 requests

Error budget = (1 - target) × total events
Error budget = 0.001 × 1,000,000
Error budget = 1,000 failed requests

You can have up to 1,000 failures in 30 days and still meet your SLO.
```

### Budget Remaining

**Budget remaining** shows how much of your error budget is left:

| Budget Remaining | Interpretation |
|------------------|----------------|
| > 50% | Healthy—room to take risks |
| 20-50% | Normal consumption |
| 10-20% | Caution—reduce risky changes |
| < 10% | Critical—focus on reliability |
| 0% | SLO breached for this period |

### Why Error Budgets Matter

Error budgets enable **data-driven tradeoffs:**

- **Budget available:** "We have room to ship that experimental feature"
- **Budget low:** "Let's delay the migration until reliability improves"
- **Budget spent:** "All hands on reliability—no new features this sprint"

This replaces subjective arguments ("Is it stable enough?") with objective measures.

---

## Burn Rate Explained

**Burn rate** measures how fast you're consuming your error budget.

### Normal Burn Rate

If your error budget should last 30 days, a burn rate of 1.0x means you'll exactly exhaust it in 30 days.

| Burn Rate | Meaning |
|-----------|---------|
| < 1x | Using budget slower than expected—healthy |
| 1x | On track to use exactly the budget |
| 2x | Will exhaust budget in 15 days instead of 30 |
| 10x | Will exhaust budget in 3 days—investigate NOW |
| 100x | Will exhaust budget in hours—incident mode |

### Burn Alerts

Honeycomb can alert when burn rate exceeds a threshold:

- **Slow burn alert:** 2x burn rate over 24 hours (gradual degradation)
- **Fast burn alert:** 14x burn rate over 1 hour (acute incident)

Configure alerts to match your response capabilities. No point alerting on fast burns at 3am if no one will respond.

---

## When to Worry

### Green Flags (Healthy)
- Budget remaining > 50%
- Burn rate < 1x
- Status: Normal
- Compliance exceeds target

### Yellow Flags (Investigate Soon)
- Budget remaining 10-30%
- Burn rate 1-3x
- Recent deployment coincides with budget consumption
- Compliance dropping toward target

### Red Flags (Act Now)
- Budget remaining < 10%
- Burn rate > 10x
- Status: Triggered
- Compliance below target

---

## Investigating a Burning SLO

When an SLO is at risk, follow this workflow:

### Step 1: Understand the SLO Definition

```
Call: get_slos with slo_id

Check:
- What is the SLI measuring?
- What counts as success/failure?
- What time period is this over?
```

### Step 2: Find the Failing Events

```
Call: run_query
  VISUALIZE: COUNT
  WHERE: <SLI filter> AND <failure criteria>
  GROUP BY: time bucket

# Example: Find failing checkout requests
  WHERE: name = "POST /checkout" AND http.status_code >= 500
```

### Step 3: Use BubbleUp to Find Patterns

```
Call: run_bubbleup
  selection: failing events
  baseline: all events matching SLI

# What do failures have in common?
```

### Step 4: Drill Into Traces

```
Call: get_trace with trace_id from a failing request

# Find the root cause in the span details
```

### Step 5: Compare to Baseline

```
# Check if this is new behavior
Compare current failure rate to:
- Same time yesterday
- Same day last week
- Before most recent deploy
```

---

## Best Practices for SLOs

### Measure Close to the User

Put SLOs on your **edge service**—the one closest to users. This captures the full user experience, not just one component.

```
GOOD: SLO on api-gateway measuring end-to-end checkout latency
BAD: SLO on database measuring query latency only
```

### Design Around User Journeys

SLOs should reflect what users care about:

| User Journey | SLO |
|--------------|-----|
| "I can complete checkout" | 99.9% of POST /checkout succeed in < 3s |
| "I can view my orders" | 99.5% of GET /orders respond in < 1s |
| "Search results are relevant" | 99% of searches return results in < 500ms |

### Exclude Known/Expected Failures

Don't burn budget on things that aren't your fault:

```
# Good SLI filter excludes client errors
success_criteria: http.status_code < 500
# (400s are client mistakes, not service failures)

# Might also exclude:
# - Rate-limited requests (429)
# - Invalid credentials (401)
# - Health check endpoints
```

### Choose Achievable Targets

| Target | Downtime/month | Good For |
|--------|----------------|----------|
| 99% | 7.2 hours | Internal tools, dev environments |
| 99.5% | 3.6 hours | Non-critical services |
| 99.9% | 43 minutes | Most production services |
| 99.95% | 22 minutes | Critical user-facing features |
| 99.99% | 4.3 minutes | Payment systems, auth (very hard!) |

**Start conservative.** It's easier to tighten an SLO than to relax one.

---

## SLOs and Deployments

### Pre-Deploy Check

Before deploying:
```
Call: get_slos

Check: Do you have error budget to absorb potential issues?
- Budget > 30%: Safe to deploy
- Budget 10-30%: Deploy with caution, be ready to rollback
- Budget < 10%: Consider delaying non-critical changes
```

### Post-Deploy Monitoring

After deploying:
```
Watch for 15-30 minutes:
- Is burn rate increasing?
- Are new error patterns appearing?
- Is latency distribution changing?

If burn rate spikes > 10x: Consider rollback
```

---

## Common SLO Patterns

### Availability SLO
```
name: checkout-availability
SLI: requests where http.status_code < 500
target: 99.9%
```

### Latency SLO
```
name: checkout-latency
SLI: requests completing in < 3000ms
target: 99%
```

### Combined SLO
```
name: checkout-success
SLI: requests where http.status_code < 500 AND duration_ms < 3000
target: 99.5%
```

### Throughput SLO
```
name: order-processing
SLI: orders processed within 1 hour of creation
target: 99.9%
```

---

## MCP Tools for SLO Work

| Task | Tool |
|------|------|
| List all SLOs | `get_slos` |
| Get SLO details | `get_slos` with `slo_id` |
| Find failing events | `run_query` with failure filter |
| Analyze failures | `run_bubbleup` |
| Examine specific failure | `get_trace` |
| Check service health | `get_service_map` |

---

## Quick Reference

### SLO Terminology
- **SLI:** What you measure
- **SLO:** Your target for that measurement
- **Error budget:** Allowable failures within the SLO window
- **Burn rate:** How fast you're using the error budget
- **Compliance:** Current percentage meeting the SLI criteria

### Burn Rate Math
```
Error budget for 30-day window at 99.9% = 0.1% of requests
Burn rate 1x = using budget at sustainable pace
Burn rate 10x = will exhaust in 3 days
```

### Investigation Flow
1. Check SLO definition
2. Query for failing events
3. BubbleUp to find patterns
4. Drill into traces for root cause
5. Compare to baselines
