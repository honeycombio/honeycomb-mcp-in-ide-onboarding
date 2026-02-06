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

When the user asks to get started (e.g., "Help me get started with Honeycomb"), greet them with:

> "Welcome to Honeycomb onboarding! Let's get you started.
>
> I'll track your progress and preferences in two small files (`progress.yaml` and `my-context.yaml`) so we can pick up where we left off. Let me get those set up.
>
> **Tip:** When the permission prompts appear, select **Always allow** for each file. I'll be updating these throughout onboarding to save your progress, and it'll be smoother if you don't have to approve every edit."

**Immediately after this message**, write an update to `onboarding/progress.yaml` (set `last_session` to the current timestamp) **and** `onboarding/my-context.yaml`. This triggers Claude Code's permission prompts. The user should select "Always allow" for each file to avoid repeated permission dialogs.

**Do not continue until both file permissions are granted.** Once they are, proceed to Step 0 below.

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
> - Product manager
> - Customer support
> - Other"

Ask what they work on — this is the most important question because it determines which data we focus on first:

> "What services or products do you work on? For example: checkout, payments, search, the iOS app — whatever you touch day to day."

Save their answer to `primary_services` in `my-context.yaml`. This will be used in Step 2 to find matching datasets and in the first query to filter to their service.

Ask about their observability experience:

> "How familiar are you with observability tools?
> - **New to it** — I haven't used tools like this before
> - **Some experience** — I've used dashboards or logs, but I'm new to tracing
> - **Experienced** — I'm comfortable with distributed tracing concepts"

Optionally ask about their goals:

> "Is there anything specific you're hoping to learn or do in Honeycomb?"

Update `my-context.yaml` with their responses (role, primary_services, experience_with_observability, learning_goals).

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

**Match the user's services:** Check `primary_services` from `my-context.yaml` against the datasets and service names returned by `get_workspace_context`. If you find a match (exact or fuzzy — e.g., "checkout" matches `checkout-service`), call it out immediately:

> "I found your checkout service — it's sending data to the `checkout-service` dataset in production. That's where we'll focus."

If there are multiple matches, pick the one with the most recent data. If there are no matches, tell the user what's available and ask which one they'd like to explore.

Update `my-context.yaml` with the matched dataset/environment so all subsequent queries use it:
```yaml
primary_services: [checkout]
matched_dataset: "checkout-service"
matched_environment: "production"
```

**Example response:**

> "Your team has 3 environments set up. Production has data from 12 services. I can see your checkout service is here as `checkout-service` — let's start there."

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

**Action:** Run a query to find interesting traces. **Include the Honeycomb link to the query results so the user can see the heatmap in the UI.**

```
Call run_query with:
- Dataset: matched_dataset from my-context.yaml (fall back to the primary environment's busiest dataset)
- HEATMAP(duration_ms) + P50(duration_ms) + P95(duration_ms) + P99(duration_ms)
- Filtered to the last hour
- Grouped by name (operation) — this shows the user's own endpoints
```

> "Let's look at what's happening in your [service name] right now..."

> "We're querying both a heatmap and percentiles together. The heatmap shows the *shape* of the distribution — you can spot bimodal patterns, outlier clusters, and shifts over time. The percentiles give you concrete numbers to quote: 'P95 is 450ms' is something you can put in a Slack message. Always pair them."

**Explain (if "heatmaps" not in concepts_learned):**

> "This heatmap shows the distribution of request durations. Each row of pixels represents a time bucket. Darker colors mean more requests at that latency.
>
> See that band of requests around 200ms? That's your typical response time. But look at these scattered dots up at 2-3 seconds—those are your slow outliers. Let's investigate one.
>
> You can see this in the Honeycomb UI here: [link]"

**Try it yourself:** Open the Honeycomb link above. Now try building this query from scratch.

1. Click **New Query** in the top nav
2. In the dataset picker, select the same dataset we just queried
3. Under **VISUALIZE**, click the `+` and select `HEATMAP(duration_ms)`. Add `P95(duration_ms)` too
4. Under **WHERE**, add a filter for the time range (last 1 hour)
5. Under **GROUP BY**, add `service.name` or `http.route`
6. Click **Run Query**

You should see the same heatmap we just looked at. Try hovering over the heatmap — the UI shows exact counts and values for each pixel bucket.

**Pro tip:** In a real investigation, you wouldn't start with a broad heatmap like this. You'd check your SLOs and triggers first — they tell you which operations your team has already declared critical, and whether any are currently at risk. That narrows your search immediately. We started broad here to teach the concepts, but going forward, start from what matters most.

**Update progress.yaml:** Add `heatmaps` to `concepts_learned`.

---

### Debugging Step 2: Fetch a Trace

Help the user select a slow request from the heatmap results, then:

**Action:** Call `get_trace` with the trace_id. **Include the Honeycomb link to the trace so the user can see the waterfall view in the UI.**

**Explain (if "traces" not in concepts_learned):**

Tell the story of what happened in this trace using the actual data. **Do not just list spans.** Narrate it in plain language:

1. **Start with context from the metadata.** Look at every field in the trace — user IDs, account names, plan tiers, endpoints, request parameters, feature flags, etc. Use whatever is there to describe who was doing what. For example: "This trace shows a request from user `acme-corp` on the Enterprise plan, hitting the `/api/v1/export` endpoint to download a CSV report."

2. **Walk through what happened step by step.** Describe the journey through services in plain terms, not span jargon: "The request came into the API gateway, which routed it to the export service. The export service queried the database for the report data, then uploaded the result to S3."

3. **Then introduce the vocabulary.** After telling the story, connect it to the concepts:

> "What you're looking at is called a **trace** — it captures the full journey of a single request through your system. Each step in that journey (the API call, the database query, the S3 upload) is called a **span**. The waterfall view shows these spans as bars, with time flowing left to right. Longer bars = more time spent."

4. **Point out what matters.** Highlight the bottleneck or interesting finding in context: "Notice that the database query took 1.8 seconds out of a total 2.1 seconds — that's where almost all the time went."

**Never hallucinate details.** Only reference fields and values actually present in the trace. But use every piece of context available to make the trace relatable.

**Try it yourself:** Open the trace link above and explore the waterfall view.

1. Find the widest bar in the waterfall — that's the bottleneck we talked about
2. Click on that span to expand its details panel on the right
3. Look through the **attributes** listed — these are the fields we used to tell the story (user IDs, endpoints, status codes, etc.)
4. Try clicking on a different span to compare its attributes

Notice how each span shows its duration, service name, and operation. The indentation shows parent-child relationships — a span indented under another was called by it.

**Update progress.yaml:** Add `traces`, `spans` to `concepts_learned`.

---

### Debugging Step 3: Find the Bottleneck

Walk through the trace with the user **using plain language, grounded in the actual data.** Continue the story from Step 2 — use the same user/customer context and describe what went wrong in terms the user can relate to.

**Example** (adapt to real data — do not use this verbatim):

> "So this Enterprise customer was trying to export their report, and the whole request took 2.3 seconds. Let's see where the time went:
>
> - The API gateway routed the request in 50ms — that's fine
> - The checkout service spent 100ms validating — also fine
> - But then the payment service called Stripe, and that took **2.1 seconds**
>
> That one call to Stripe ate 91% of the total time. So the slowness isn't in your code — it's the wait on an external API."

**Tips to teach the user how to read traces on their own:**
- The widest bars in the waterfall are where the time goes — start there
- If you see spans landing at exactly 5s, 10s, 30s, or 60s, that's usually a **timeout**, not the real duration. These round numbers map to specific config values: 5s is a common HTTP client default, 10s is often a database timeout, 60s is a typical load balancer timeout. Spotting a timeout tells you *where* to look in code or config — and that the actual operation would have taken even longer
- Error spans are typically marked with `status: error` or highlighted in red

---

### Debugging Step 4: Introduce BubbleUp

> "We found ONE slow trace. But is this a pattern? Let's use **BubbleUp** to find what slow requests have in common."

**Action:** If you have a query with a heatmap, run `run_bubbleup` selecting the slow outliers. **Include the Honeycomb link to the query so the user can see the BubbleUp results in the UI.**

**Explain (if "bubbleup" not in concepts_learned):**

> "BubbleUp compares your selection (the slow requests) against the baseline (everything else). It checks every field to find what's different.
>
> For example, it might discover:
> - 80% of slow requests have `user.plan = enterprise` (vs 20% baseline)
> - 95% of slow requests hit `endpoint = /export` (vs 5% baseline)
>
> This surfaces correlations you'd never think to check manually."

**After reviewing BubbleUp results, validate the top finding:**

> "BubbleUp said 80% of slow requests have `user.plan = enterprise`. But is that meaningful? Let's check the **base rate** — what percentage of *all* requests are enterprise?"

**Action:** Run a COUNT query grouped by the correlated field (e.g., `user.plan`) without filtering to the slow subset. **Include the Honeycomb link to the query results.**

> "If 75% of all traffic is enterprise, then 80% in the slow group is barely elevated — that's a lift of only 1.07x. But if only 20% of all traffic is enterprise, then 80% in the slow group is a 4x lift — that's a strong signal.
>
> **Lift = (rate in selection) / (rate in baseline).** A lift under 1.5x is weak. Always check before drawing conclusions."

**Try it yourself:** Open the query link from Step 1 and try running BubbleUp in the UI.

1. On the heatmap, click and drag to select a region of slow outliers (the scattered dots at the top)
2. A **BubbleUp** panel will appear below the chart
3. Scroll through the results — each row shows a field where the selection differs from the baseline
4. Look for the **Baseline** vs **Selection** percentage bars. Fields where the bars look very different have high lift

Compare what you see in the UI to what I showed you. The bar chart visualization makes it easier to spot strong signals at a glance than reading the numbers.

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

**Action:** Run a simple aggregation query. **Include the Honeycomb link to the query results so the user can see them in the UI.**

```
DATASET: matched_dataset from my-context.yaml (fall back to the primary environment's busiest dataset)
VISUALIZE: COUNT, P95(duration_ms)
WHERE: service.name = <user's primary service from my-context.yaml>
GROUP BY: name
TIMERANGE: last 1 hour
```

> "Let's query your [service name] and see what operations it handles..."

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

**A note on GROUP BY hygiene:** We filtered to a single service above, but notice we're also grouping by `name` (the span/operation name). That's intentional — if you GROUP BY a field like `user.tier` without filtering to a specific operation first, you'd be mixing health checks (1ms) with exports (30s) in the same buckets, and the numbers would be meaningless. **Always filter to a specific span name before grouping by other fields.**

**Try it yourself:** Open the query link above and try modifying it.

1. In the query builder, change `P95(duration_ms)` to `P99(duration_ms)` — click on the calculation and edit it
2. Add a new WHERE filter: click `+` under WHERE and filter to a specific `name` (operation)
3. Click **Run Query** to see how the results change
4. Try adding `HEATMAP(duration_ms)` to VISUALIZE alongside the percentile — now you get both the distribution shape and the numbers

The Query Builder is where you'll spend most of your time in Honeycomb. Every field you see in the dropdowns comes from your instrumentation data.

**Update progress.yaml:** Add `queries`, `percentiles` to `concepts_learned`.

**Celebrate:**

> "You just ran your first Honeycomb query! You can slice this data any way you want—by user, by endpoint, by region, by error type."

---

### Exploration Step 4: Discover SLOs

**Action:** Call `get_slos` for the environment. **Include the Honeycomb link to the SLOs page so the user can explore them in the UI.**

> "Your team has set up **SLOs** (Service Level Objectives) to define what 'good' looks like. Let's see what they're tracking..."

Summarize the SLOs found and include a direct link:

> "Here are the SLOs your team is tracking — you can see them all in the Honeycomb UI here: [link]"

**Also check triggers:** Call `get_triggers` for the environment. If triggers exist, summarize them:

> "Your team also has **triggers** set up — these are alerts that fire when a metric crosses a threshold. You can see them here: [link]"

If no SLOs exist:

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

**Action:** Call `get_slos` to fetch current SLOs. **Include the Honeycomb link to the SLOs page.**

**Also check triggers:** Call `get_triggers` for the environment. Triggers are alerts tied to specific queries — they're often the first signal that something is wrong.

**Explain (if "slos" not in concepts_learned):**

> "An **SLO** (Service Level Objective) is a target for how reliable your service should be. For example: '99.9% of checkout requests should succeed within 500ms.'
>
> Each SLO has:
> - **SLI** (Service Level Indicator) — The metric being measured (success rate, latency, etc.)
> - **Target** — The goal (99.9%, 99.5%, etc.)
> - **Time window** — The period over which it's measured (30 days, rolling)
>
> When you're meeting your SLO, everything is fine. When you're not, it's time to investigate.
>
> You can see your SLOs in the Honeycomb UI here: [link]"

If triggers were found:

> "Your team also has **triggers** — these fire alerts when a metric crosses a threshold (e.g., 'P95 latency > 500ms for 5 minutes'). You can see them here: [link]"

**Try it yourself:** Find the SLOs page in the Honeycomb UI.

1. In the left sidebar, click **SLOs**
2. You'll see the list of SLOs we just reviewed — find one and click into it
3. Look at the **Budget Burndown** graph — it shows how error budget is being consumed over time
4. Check the **Burn Alerts** section at the bottom — these fire when the budget is burning too fast

If an SLO shows a declining budget line, click **View SLI** to jump to the underlying query that defines what "good" means.

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

**Action:** **Include Honeycomb links for every query and trace in this investigation.**
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
> From the SLO view → Click into failing events → Select a trace → See exactly what went wrong.
>
> Here's the SLO in the Honeycomb UI: [link]"

**Action:** Demonstrate by fetching a trace from a failing request. **Include the Honeycomb link to the trace.** When explaining the trace, narrate the full story using the metadata — who the user was, what they were doing, and what went wrong — following the same approach as Debugging Step 2.

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
> "A **trace** captures the full story of a single request — who made it, what they were trying to do, and every step your system took to handle it. Each step is called a **span** (an API call, a database query, a cache lookup). Spans nest inside each other: when one service calls another, the callee's span is a child of the caller's span."
>
> When explaining traces, always narrate the story using the actual metadata (user info, endpoints, request details) before introducing vocabulary. Never assume users can read a span waterfall without guidance.

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
