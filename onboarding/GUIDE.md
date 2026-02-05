# Honeycomb Onboarding Guide

This guide helps new engineers learn Honeycomb through their actual production data. It adapts to experience level and tracks progress to avoid repetition.

---

## Entry Logic

Before starting, check `progress.yaml` in this folder:

```
IF started: false
  → Begin WELCOME FLOW below

IF started: true AND completed: false
  → Resume from current_step value

IF completed: true
  → Skip onboarding, be concise, follow investigation/GUIDE.md for debugging
```

---

## Welcome Flow

### Step 0: Verify MCP Connection (Pre-flight Check)

**Before starting onboarding, check if Honeycomb MCP is connected.**

**Detection:** Run the `/mcp` command to check MCP status.
- If authenticated to Honeycomb → Proceed to Step 1
- If needs authentication → Tell user to authenticate via `/mcp`
- If not configured → Follow setup flow below

**If MCP is NOT configured:**
- Follow the setup flow in `setup-mcp.md`
- Run: `claude mcp add honeycomb --transport http https://mcp.honeycomb.io/mcp`
- Tell user to exit, restart, and run `/mcp` to authenticate
- Wait for them to return after authentication

**If MCP IS connected:**
- Proceed to Step 1 below

**Update progress.yaml:**
```yaml
mcp_connected: true
```

---

### Step 1: Get to Know the User

**Before exploring any data, learn about the user so the experience can be tailored to them.**

If `my-context.yaml` has null values, ask the user about themselves:

> "Before we dive in, I'd love to learn a bit about you so I can tailor this to your needs."

Ask about their role:

> "What's your role?
> - Product engineer (building features)
> - SRE / Platform (reliability & infrastructure)
> - Data / Analytics
> - Manager / Leadership
> - Other"

Ask about their observability experience:

> "How familiar are you with observability tools?
> - **New to it** — I haven't used tools like this before
> - **Some experience** — I've used dashboards or logs, but I'm new to tracing
> - **Experienced** — I'm comfortable with distributed tracing concepts"

Optionally ask about their goals:

> "Is there anything specific you're hoping to learn or do in Honeycomb?"

Update `my-context.yaml` with their responses (role, experience_with_observability, learning_goals).

**Update progress.yaml:**
```yaml
started: true
current_step: "workspace_discovery"
first_session: <current_timestamp>
```

---

### Step 2: Discover What's Instrumented

**Action:** Call `get_workspace_context` via the Honeycomb MCP.

**Then explain to the user (calibrated to their experience level from Step 1):**

> "Let me see what's flowing into your Honeycomb account..."

After the call returns, be concise and summarize:
- How many environments exist (production, staging, dev, etc.)
- Which datasets/services are sending data
- Approximate data volume if visible

**Example response:**

> "Your team has 3 environments set up. Production has data from 12 services including `api-gateway`, `checkout-service`, and `user-auth`."

If the user mentioned primary services or learning goals in Step 1, connect what you find to their interests (e.g., "I can see your checkout service is instrumented — we can explore that together").

**Update progress.yaml:**
```yaml
current_step: "path_selection"
```

---

### Step 3: Choose a Learning Path

Ask the user what brings them here:

> "What would you like to do today?
> 1. **I'm debugging something** — I have a specific issue to investigate
> 2. **Just exploring** — I want to understand what's here and how it works
> 3. **Checking reliability** — I want to see how our services are performing"

Based on their answer, follow the corresponding path below.

If the user shared learning goals in Step 1, you can suggest a path:
> "Since you mentioned wanting to understand traces, the exploration path might be a great fit."

**Update progress.yaml:**
```yaml
path: "debugging" | "exploration" | "reliability"
current_step: "<path>_step_1"
```

---

## Path A: Debugging (Learn Traces)

### Debugging Step 1: Find Something to Investigate

**Action:** Run a query to find interesting traces.

```
Call run_query with:
- A heatmap of duration_ms
- Filtered to the last hour
- Grouped by service.name or http.route
```

**Explain (if "heatmaps" not in concepts_learned):**

> "This heatmap shows the distribution of request durations. Each row of pixels represents a time bucket. Darker colors mean more requests at that latency.
>
> See that band of requests around 200ms? That's your typical response time. But look at these scattered dots up at 2-3 seconds—those are your slow outliers. Let's investigate one."

**Update progress.yaml:** Add `heatmaps` to `concepts_learned`.

---

### Debugging Step 2: Fetch a Trace

Help the user select a slow request from the heatmap results, then:

**Action:** Call `get_trace` with the trace_id.

**Explain (if "traces" not in concepts_learned):**

> "This is a **trace**—a visualization of everything that happened to handle one request. Think of it like a receipt showing every step your code took.
>
> The waterfall shows time flowing left to right. Each bar is called a **span**—one unit of work. Spans can be nested: a parent span (like 'handle HTTP request') contains child spans (like 'query database', 'call external API').
>
> Key fields to notice:
> - **duration_ms** — How long this span took
> - **name** — What operation this represents
> - **service.name** — Which service executed this code
> - **status** — Whether it succeeded or errored"

**Update progress.yaml:** Add `traces`, `spans` to `concepts_learned`.

---

### Debugging Step 3: Find the Bottleneck

Walk through the trace with the user:

> "Let's read this trace together. The total request took 2.3 seconds. Scanning down...
>
> - `api-gateway` spent 50ms routing
> - `checkout-service` spent 100ms validating
> - `payment-service` → `stripe-api` call took **2.1 seconds** ← There's our culprit
>
> 91% of the time was spent waiting for the Stripe API. That's either a slow external dependency or a network issue."

**Tips to share:**
- Look for the widest bars—they dominate total time
- Fixed round durations (5s, 10s, 30s, 60s) often indicate timeouts
- Error spans are usually marked with `status: error` or colored red

---

### Debugging Step 4: Introduce BubbleUp

> "We found ONE slow trace. But is this a pattern? Let's use **BubbleUp** to find what slow requests have in common."

**Action:** If you have a query with a heatmap, run `run_bubbleup` selecting the slow outliers.

**Explain (if "bubbleup" not in concepts_learned):**

> "BubbleUp compares your selection (the slow requests) against the baseline (everything else). It checks every field to find what's different.
>
> For example, it might discover:
> - 80% of slow requests have `user.plan = enterprise` (vs 20% baseline)
> - 95% of slow requests hit `endpoint = /export` (vs 5% baseline)
>
> This surfaces correlations you'd never think to check manually."

**Update progress.yaml:** Add `bubbleup` to `concepts_learned`.

**Celebrate:**

> "You just completed your first investigation! You went from 'something is slow' to 'enterprise users hitting /export are slow because of Stripe API latency.' That's the core analysis loop in Honeycomb."

---

### Debugging Step 5: Wrap Up

**Update progress.yaml:**
```yaml
current_step: "debugging_complete"
```

> "Want to try something else?
> - Explore your SLOs to see what your team considers 'reliable'
> - Look at another service's data
> - Learn how to set up alerts for slow requests"

If user has completed traces, spans, heatmaps, bubbleup, and queries → set `completed: true`.

---

## Path B: Exploration (Learn the Data Model)

### Exploration Step 1: Environments and Datasets

**Action:** Call `get_workspace_context` then `get_environment` for the primary environment.

**Explain (if "data_model" not in concepts_learned):**

> "Honeycomb organizes data in a hierarchy:
>
> **Team** → Your Honeycomb account
> **Environments** → Usually maps to deployment stages (prod, staging, dev)
> **Datasets** → Usually one per service or application
>
> Each dataset contains **events**—structured records of things that happened. When you instrument with OpenTelemetry, each span becomes an event."

**Update progress.yaml:** Add `data_model` to `concepts_learned`.

---

### Exploration Step 2: Understanding Columns

**Action:** Call `get_dataset_columns` for a relevant dataset.

> "Every event has **columns** (also called fields or attributes). These are the dimensions you can query, filter, and group by.
>
> Common columns you'll see:
> - `duration_ms` — How long the operation took
> - `name` — The operation name (HTTP route, function name, etc.)
> - `service.name` — Which service generated this event
> - `http.status_code` — Response codes for HTTP requests
> - `trace.trace_id` / `trace.span_id` — IDs linking spans into traces
>
> Your instrumentation determines what columns exist. More fields = more ways to slice your data."

---

### Exploration Step 3: Run Your First Query

**Action:** Run a simple aggregation query:

```
VISUALIZE: COUNT, P95(duration_ms)
WHERE: service.name = <a real service>
GROUP BY: name
TIMERANGE: last 1 hour
```

**Explain (if "queries" not in concepts_learned):**

> "Queries in Honeycomb have a few key parts:
>
> - **VISUALIZE** — What calculation to perform (COUNT, AVG, P95, HEATMAP, etc.)
> - **WHERE** — Filter to specific events
> - **GROUP BY** — Break results into buckets by field values
>
> This query shows 'for each operation name, how many requests and what's the 95th percentile latency?'
>
> **P95 means:** 95% of requests were faster than this value. It's more useful than averages because averages hide outliers."

**Update progress.yaml:** Add `queries`, `percentiles` to `concepts_learned`.

**Celebrate:**

> "You just ran your first Honeycomb query! You can slice this data any way you want—by user, by endpoint, by region, by error type."

---

### Exploration Step 4: Discover SLOs

**Action:** Call `get_slos` for the environment.

> "Your team has set up **SLOs** (Service Level Objectives) to define what 'good' looks like. Let's see what they're tracking..."

Summarize the SLOs found. If none exist:

> "No SLOs are configured yet. SLOs are a way to say 'we promise 99.9% of requests will succeed' and track that automatically. Want me to explain how they work?"

---

### Exploration Step 5: Wrap Up

> "You now understand how Honeycomb organizes data. The next step is usually:
> - Investigate a specific slow request (debugging path)
> - Understand your SLOs and error budgets (reliability path)
> - Explore the service map to see how services connect"

Update progress.yaml with concepts learned.

---

## Path C: Reliability (Learn SLOs)

### Reliability Step 1: What Are SLOs?

**Action:** Call `get_slos` to fetch current SLOs.

**Explain (if "slos" not in concepts_learned):**

> "An **SLO** (Service Level Objective) is a target for how reliable your service should be. For example: '99.9% of checkout requests should succeed within 500ms.'
>
> Each SLO has:
> - **SLI** (Service Level Indicator) — The metric being measured (success rate, latency, etc.)
> - **Target** — The goal (99.9%, 99.5%, etc.)
> - **Time window** — The period over which it's measured (30 days, rolling)
>
> When you're meeting your SLO, everything is fine. When you're not, it's time to investigate."

**Update progress.yaml:** Add `slos`, `slis` to `concepts_learned`.

---

### Reliability Step 2: Understanding Error Budgets

**Explain (if "error_budgets" not in concepts_learned):**

> "The **error budget** is how much failure you can tolerate while still meeting your SLO.
>
> If your target is 99.9% over 30 days and you have 1 million requests:
> - You can have up to 1,000 failed requests (0.1% of 1M)
> - That's your error budget
>
> **Burn rate** is how fast you're using up that budget:
> - Normal: Using budget at expected rate
> - Burning fast: Using budget faster than sustainable—investigate!
> - Healthy: Using less budget than expected
>
> Error budgets help you make tradeoffs: 'We have budget to spare—let's ship that risky feature' vs 'We're burning too fast—time to focus on reliability.'"

**Update progress.yaml:** Add `error_budgets`, `burn_rate` to `concepts_learned`.

---

### Reliability Step 3: Investigating a Burning SLO

If any SLO is in a concerning state (triggered or burning fast):

**Action:**
1. Note which SLO is at risk
2. Run a query filtered to failing events
3. Use BubbleUp to find correlations

> "Your 'checkout-success' SLO is burning budget faster than normal. Let's investigate...
>
> I'll query for checkout requests that failed and use BubbleUp to find patterns."

Walk through the investigation using the debugging techniques from Path A.

---

### Reliability Step 4: Connecting SLOs to Traces

> "When an SLO is at risk, you can drill down to individual traces to find root causes.
>
> From the SLO view → Click into failing events → Select a trace → See exactly what went wrong."

**Action:** Demonstrate by fetching a trace from a failing request.

---

### Reliability Step 5: Wrap Up

> "You now understand how SLOs work in Honeycomb:
> - SLIs measure what matters
> - SLOs set targets
> - Error budgets quantify acceptable failure
> - Burn rates show if you need to act
>
> Want to explore more?
> - Set up a burn alert to get notified when budget is at risk
> - Compare SLO performance across time periods
> - Investigate why a specific SLO is burning"

Update progress.yaml with concepts learned.

---

## Concept Explanations Reference

Only explain these when the concept is NOT in `concepts_learned`:

### Traces and Spans
> "A **trace** shows the complete journey of a single request through your system. Each step in that journey is a **span**. Spans have parent-child relationships—when service A calls service B, B's span is a child of A's span. The trace ID links them all together."

### Heatmaps
> "A **heatmap** shows distribution over time. The Y-axis is the value (like latency), X-axis is time, and color intensity shows count. It reveals patterns that line charts hide—like bimodal distributions or rare outliers."

### Percentiles (P50, P95, P99)
> "**P50** (median) — Half of requests were faster, half slower
> **P95** — 95% of requests were faster than this
> **P99** — 99% of requests were faster than this
>
> P95/P99 matter more than averages because they show what your slowest users experience."

### BubbleUp
> "**BubbleUp** automatically compares two sets of data and finds what's different. Select anomalous data (slow requests, errors), and it shows which field values are over-represented compared to the baseline."

### Baselines
> "Always compare against **baselines** to understand if something is actually anomalous:
> - vs. previous hour
> - vs. same time yesterday
> - vs. same day last week
>
> A P99 of 500ms might be normal or terrible—depends on the baseline."

---

## Progress Tracking

After each teaching moment, update `progress.yaml`:

```yaml
started: true
current_step: "<current_step_name>"
completed: false  # Set true when core concepts learned
path: "debugging" | "exploration" | "reliability"
concepts_learned:
  - traces
  - spans
  - heatmaps
  - queries
  - percentiles
  - slos
  - slis
  - error_budgets
  - burn_rate
  - bubbleup
  - baselines
  - data_model
last_session: <current_timestamp>
```

**Mark completed: true** when user has learned at minimum:
- traces, spans
- queries (VISUALIZE, WHERE, GROUP BY)
- One of: SLOs OR BubbleUp

---

## Tone Reminders

- Friendly, not overwhelming
- Teach through their actual data
- Celebrate small wins
- Always offer a next step
- Acknowledge when something is complex: "This takes practice to read at a glance"
