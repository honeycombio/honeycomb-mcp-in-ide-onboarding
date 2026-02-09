# Analysis Rules — Always Applied

These rules apply silently to every MCP query and analysis. Do not explain them unless the user asks why you did something. Just follow them.

---

## R1: Baselines

When identifying anomalies, compare against at least two baselines:

- Previous hour
- Same time yesterday
- Same day last week
- Same date last month (catches monthly billing cycles, batch jobs, report generation)
- Same day-of-week last month (catches weekly patterns that shift with calendar dates)

State the comparison explicitly: "P95 is 450ms vs 220ms same time yesterday — a 2x increase."

## R2: Heatmap + Percentiles

Every latency query must include both `HEATMAP(duration_ms)` and `P50(duration_ms)`, `P95(duration_ms)`, `P99(duration_ms)`. The heatmap reveals distribution shape; the percentiles give concrete numbers. Never use only `AVG(duration_ms)` for latency.

## R3: Filter Before Grouping

Before any `GROUP BY` analysis, check whether the dataset contains multiple span `name` values. If it does, filter to a single operation first. Grouping across mixed operations (health checks + exports + queries) produces meaningless aggregates.

## R4: Critical Spans

Before exploring broadly, check SLOs (`get_slos`) and triggers (`get_triggers`) to identify the 1–2 operations the team has already declared critical. Start investigations there — these spans have defined "good" thresholds and are what the team cares about most.

## R5: BubbleUp Validation

After every BubbleUp result, check the base rate of the top-correlated field. Report **lift** — the ratio of the field's rate in the selection to its rate in the baseline:

```
lift = (% in selection) / (% in baseline)
```

A lift under 1.5x is weak signal. Only highlight findings with meaningful lift.

## R6: Timeout Detection

When heatmaps or percentiles show clusters at round numbers (5s, 10s, 30s, 60s, 120s), flag them as probable timeouts. These map to specific code or infrastructure config values — note what the likely source is (HTTP client, database, load balancer, Lambda). The timeout masks the real duration; the underlying operation would have taken longer.

## R7: Rare Blockers

For queue or concurrency investigations, sort by `MAX(duration_ms)` descending and look for operations with **high duration + low count**. The rare long-running operation is usually the blocker — not the piled-up fast operations waiting behind it.

## R8: No Averages on Small Samples

Never use AVG on a small sample of data. Averages are only meaningful with enough volume to smooth out outliers. If a GROUP BY bucket has fewer than ~30 events, report the raw values or use COUNT instead. One 10-second request in a group of 3 produces a misleading AVG. Prefer percentiles (P50, P95, P99) which degrade more gracefully with low volume — or just note that the sample is too small to draw conclusions.

## R9: Know When to Pivot

If three rounds of queries and BubbleUp produce no clear signal, tell the user directly. The problem may not be visible in Honeycomb — suggest checking: infrastructure metrics, external dependency status pages, client-side instrumentation, network/DNS, or uninstrumented code paths.

## R10: Structured First, Logs Too

Always start analysis with structured queries — aggregates (COUNT, P95, error rates), breakdowns (GROUP BY status code, endpoint, service), and heatmaps. These reveal patterns across thousands of events that no amount of log reading can surface.

When a user asks for "logs" or wants to search for error messages, **do both**:

1. **Show them what they asked for.** Run the query that surfaces matching events or samples. Respect their workflow — they may have good reasons for wanting to see raw output.
2. **Also show the structured view.** Run a complementary query that answers the same question with aggregates, traces, or BubbleUp. Frame it as additive: "Here are the matching events. I also ran a breakdown by status code — it shows 92% of these errors are coming from one endpoint, which might help narrow this down."
3. **Connect to traces when relevant.** If the events have trace IDs, offer to pull a representative trace: "Want me to grab the trace for one of these? It'll show the full request path that led to this error."

The goal is to meet users where they are and expand their toolkit — not to block log-based workflows. Over time, users who see structured queries alongside their logs will naturally start reaching for the structured approach first.
