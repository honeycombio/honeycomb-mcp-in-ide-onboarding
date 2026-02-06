# Investigation Guide: Expert Debugging Heuristics

This guide encodes expert debugging heuristics for investigating issues in Honeycomb. Use these patterns when helping users debug production problems.

---

## The Core Analysis Loop

1. **Start broad** — Query for anomalies (errors, slow requests, unusual patterns)
2. **Identify outliers** — Use heatmaps to spot unusual distributions
3. **BubbleUp** — Compare anomalies to baseline to find correlations
4. **Drill into traces** — Examine individual requests to confirm hypotheses
5. **Iterate** — Refine queries based on what you learn

---

## Baseline Comparisons

**Always compare against baselines.** An anomaly is only meaningful relative to normal behavior.

### Recommended Baselines

| Baseline | When to Use | MCP Approach |
|----------|-------------|--------------|
| Previous hour | Immediate comparison | Query with time offset |
| Same time yesterday | Account for daily patterns | Compare 24h ago |
| Same day last week | Account for weekly patterns | Compare 7d ago |
| Same date last month | Catch monthly patterns | Compare ~30d ago |
| Same day-of-week last month | Weekly pattern + calendar shift | Compare ~28d ago |
| Pre-deploy baseline | After a deployment | Filter by version/commit |

### How to Compare

```
# Current period
run_query: last 1 hour

# Compare to same time yesterday
run_query: 25 hours ago to 24 hours ago

# Look for: COUNT changes, P95 changes, error rate changes
```

**Monthly patterns to watch for:**
- Billing cycle runs (1st/15th of month)
- Scheduled batch jobs or report generation
- License renewal or quota reset spikes

**Red flags:**
- P95 doubled compared to yesterday
- Error rate up 10x from baseline
- Request count dropped suddenly (might indicate upstream failure)

---

## Heatmap Interpretation

**Always use heatmaps for latency.** Line charts of averages hide the distribution.

### What to Look For

| Pattern | What It Means | Next Step |
|---------|---------------|-----------|
| Single band | Consistent latency | Normal—nothing to investigate |
| Two bands | Bimodal distribution | GROUP BY to find the split (cache hit/miss? different endpoints?) |
| Scattered dots above | Slow outliers | BubbleUp to find what's different |
| Band shifting up | Latency regression | Check recent deploys, compare to baseline |
| Band widening | Increased variance | Look for resource contention |

### Bimodal Distributions

When you see two distinct latency bands:

```
run_query:
  VISUALIZE: HEATMAP(duration_ms)
  GROUP BY: <suspect field>

# Common causes of bimodal latency:
# - Cache hit vs miss
# - Different endpoints (fast health check vs slow export)
# - Different user tiers (free vs enterprise)
# - Different regions (nearby vs far)
```

---

## GROUP BY Best Practices

### Check for Multiple Span Names First

When grouping by a field, verify you're not mixing different operations:

```
# WRONG: Mixing all span types
run_query:
  VISUALIZE: P95(duration_ms)
  GROUP BY: user.tier

# RIGHT: Filter to specific operation first
run_query:
  VISUALIZE: P95(duration_ms)
  WHERE: name = "POST /checkout"
  GROUP BY: user.tier
```

**Why this matters:** A query like "P95 by user tier" is meaningless if it's averaging health checks (1ms) with exports (30s).

### Useful GROUP BY Fields

| Field | When to Use |
|-------|-------------|
| `name` or `http.route` | Identify which operations are slow |
| `service.name` | Compare performance across services |
| `http.status_code` | Separate successes from errors |
| `error` | Boolean split: failures vs successes |
| `user.id` or `user.tier` | Find user-specific issues |
| `region` or `availability_zone` | Identify infrastructure problems |
| `version` or `deploy.id` | Compare across releases |

### Watch for Conflicting Field Names Across Spans

The same column name can mean different things in different span types. For example, `status` might be an HTTP status code on an API span but a job state (`queued`, `running`, `done`) on a worker span.

**Before trusting a GROUP BY result**, confirm the field has consistent meaning across the spans in your query. If you're querying across span types, filter to a single `name` first (R3) or inspect sample events to verify the field semantics match.

---

## BubbleUp Validation

BubbleUp finds correlations, but **correlation isn't causation.** Validate findings before concluding.

### Check Base Rates

When BubbleUp shows a high proportion:

```
"80% of slow requests have user.tier = enterprise"
```

**Before concluding enterprise users are slow, check:**
- What % of ALL requests are enterprise?
- If 75% of all traffic is enterprise, then 80% in slow requests is barely elevated
- Look for **lift** (ratio of selection rate to baseline rate), not just absolute %

### Validate with Targeted Queries

After BubbleUp suggests a correlation:

```
# BubbleUp says: slow requests often have endpoint = /export

# Validate: Is /export actually slower?
run_query:
  VISUALIZE: P95(duration_ms)
  GROUP BY: http.route
  WHERE: http.route in ["/export", "/api/users", "/checkout"]

# If /export is 10x slower than other endpoints, correlation is real
```

### Multiple Correlated Fields

When BubbleUp highlights multiple fields, they might be correlated:

```
# BubbleUp shows:
# - user.tier = enterprise (elevated)
# - payload_size > 10MB (elevated)

# These might be the same thing: enterprise users upload big files
# Test by controlling for one variable:

run_query:
  VISUALIZE: P95(duration_ms)
  WHERE: payload_size > 10MB
  GROUP BY: user.tier

# If enterprise is still elevated, it's not just payload size
```

---

## Timeout Detection

**Fixed durations often indicate timeouts.**

### Common Timeout Values

| Duration | Likely Source |
|----------|---------------|
| 5s | Default HTTP client timeout |
| 10s | Database query timeout |
| 30s | AWS Lambda max (older) |
| 60s | Load balancer timeout, many defaults |
| 120s | Nginx default proxy timeout |
| 300s | Some background job timeouts |

### What to Do

When you see requests clustering at exact round numbers:

```
# Query for requests near timeout boundary
run_query:
  VISUALIZE: COUNT
  WHERE: duration_ms > 9500 AND duration_ms < 10500
  GROUP BY: name, service.name

# These are likely hitting a 10s timeout
# Next: Check code/config for where this timeout is set
```

**Timeouts mask the real problem.** A 10s timeout means the actual operation would have taken longer—find out why.

---

## Queue and Concurrency Issues

### Signs of Queuing

- Latency increases with throughput (more requests = slower)
- P50 stays stable while P95/P99 increase
- Sawtooth patterns in latency graphs

### Rare Long Tasks Blocking

**Don't just look at high-volume operations.** A rare slow task can block many fast ones. The intuition: when investigating queuing, the piled-up fast operations are *victims*, not the cause. Look for the blocker.

```
# Find the blocker: sort by MAX duration, look for low COUNT + high duration
run_query:
  VISUALIZE: COUNT, MAX(duration_ms), P99(duration_ms)
  GROUP BY: name
  ORDER BY: MAX(duration_ms) DESC
  TIMERANGE: last 1 hour

# What to look for in results:
# - name="db.migrate"  COUNT=3   MAX=62000ms  ← THIS is the blocker
# - name="db.query"    COUNT=8400 MAX=61500ms  ← These are victims (queued behind migrate)
# - name="db.insert"   COUNT=2100 MAX=60800ms  ← Also victims
#
# The victims have high MAX too (they waited), but their COUNT is high.
# The blocker has low COUNT but high MAX — it's the rare operation holding the lock.
```

**Key pattern:** Sort by `MAX(duration_ms)` DESC. The blocker is the row with **high duration + low count**. The victims have high duration + high count (many operations queued behind the blocker).

### Worker Pool Exhaustion

When all workers are busy, new requests queue:

```
# Look for correlation between concurrent requests and latency
run_query:
  VISUALIZE: HEATMAP(duration_ms), COUNT
  GROUP BY: <time bucket>

# Spikes in count that correlate with latency bands shifting up = likely queuing
```

---

## Critical Spans: Start from SLOs and Triggers

Before exploring broadly, check what the team has already declared important.

### Why Start Here

SLOs and triggers encode institutional knowledge — the team has already decided which operations matter and what "good" looks like. Starting from these gives you:

- A defined threshold for "bad" (the SLO target or trigger condition)
- A narrowed scope (specific dataset, specific filter)
- Context the team cares about (not just any anomaly — a relevant one)

### How to Find Critical Spans

```
# 1. Check SLOs — these define the operations with reliability targets
get_slos → note the SLI definitions (dataset, filter, success criteria)

# 2. Check triggers — these define conditions the team wants alerts for
get_triggers → note the query conditions and thresholds

# 3. Start your investigation scoped to a critical span
run_query:
  VISUALIZE: HEATMAP(duration_ms), P50(duration_ms), P95(duration_ms), P99(duration_ms)
  WHERE: <SLO/trigger filter conditions>
  TIMERANGE: last 2 hours
```

### Example Workflow

1. `get_slos` reveals a "checkout-latency" SLO targeting P99 < 500ms
2. The SLO is burning budget — P99 is at 850ms
3. Start with `run_query` filtered to checkout spans, not a broad dataset scan
4. Use BubbleUp on the slow checkout requests specifically
5. Drill into traces from the failing population

This is faster and more focused than starting with "show me everything."

---

## When Signal Isn't in Honeycomb

Sometimes the answer isn't in your traces. Check these:

### External Dependencies Not Instrumented

```
# You see: 5s spent in "call external API" span
# But no child spans showing what happened inside

# The problem might be:
# - DNS resolution (not usually traced)
# - TLS handshake (not usually traced)
# - Network latency to third party
# - Third party being slow (you only see the wait time)
```

**What to do:** Check external service status pages, network metrics, or add more detailed instrumentation.

### Infrastructure Issues

| Symptom | Might Be |
|---------|----------|
| All services slow at same time | Infrastructure (network, disk, node pressure) |
| One AZ slow | Regional infrastructure issue |
| Periodic latency spikes | Garbage collection, cron jobs, backups |
| Sudden drop in request count | Upstream failure, load balancer issue |

**What to do:** Check infrastructure dashboards, Kubernetes metrics, cloud provider status.

### Client-Side Issues

If users report problems but server traces look fine:

- Client network issues
- Client-side processing (JavaScript, mobile app)
- CDN or edge caching problems
- DNS or certificate issues

---

## Investigation Checklist

When debugging an issue, work through these in order:

### 1. Define the Problem
- [ ] What's the symptom? (Errors? Latency? Dropped requests?)
- [ ] When did it start? (Check for deploy correlation)
- [ ] Who's affected? (All users? Specific segment?)

### 2. Establish Baseline
- [ ] What's normal for this metric?
- [ ] Compare to same time yesterday
- [ ] Compare to same day last week

### 3. Narrow the Scope
- [ ] Which service(s) are affected?
- [ ] Which endpoints/operations?
- [ ] Filter to specific span `name` before further analysis

### 4. Find Correlations
- [ ] Run BubbleUp on anomalous data
- [ ] Validate base rates for any correlations found
- [ ] Check for confounding variables

### 5. Examine Traces
- [ ] Get trace for representative bad request
- [ ] Find the bottleneck span
- [ ] Check for timeout signatures (round numbers)
- [ ] Look for error details in span attributes

### 6. Confirm Hypothesis
- [ ] Can you reproduce with a targeted query?
- [ ] Does fixing the suspected cause help?
- [ ] Does the pattern match what you'd expect?

---

## MCP Tools Reference

| Task | Tool | Example |
|------|------|---------|
| Start investigation | `get_workspace_context` | Understand what's instrumented |
| Find anomalies | `run_query` with HEATMAP | Visual distribution analysis |
| Find correlations | `run_bubbleup` | Compare selection to baseline |
| Examine one request | `get_trace` | Drill into specific trace ID |
| Check SLO status | `get_slos` | See if reliability is at risk |
| Discover fields | `find_columns` | Find what attributes exist |
| See service relationships | `get_service_map` | Understand dependencies |

**Always include a Honeycomb UI link** when presenting query results, traces, boards, or SLOs so the user can see the data themselves.

**When presenting traces**, narrate the story using the actual metadata — who the user was, what they were doing, which services were involved, and what went wrong. Do not just list spans and durations. Use every available field (user IDs, account names, endpoints, request parameters, feature flags) to paint the full picture. Never hallucinate details not present in the data.

---

## Anti-Patterns to Avoid

### Don't Average Everything
Averages hide outliers. Use heatmaps and percentiles.

### Don't Mix Operations
Always filter to specific span names before GROUP BY analysis.

### Don't Assume Correlation = Causation
Validate BubbleUp findings with targeted queries.

### Don't Ignore Low-Volume High-Impact
A rare 60-second query can cause more pain than 1000 fast ones.

### Don't Stop at the First Answer
The first correlation you find might be a symptom, not the cause. Keep digging.
