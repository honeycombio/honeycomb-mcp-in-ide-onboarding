# Investigate: Honeycomb Debugging Workflow

Run the core observability investigation loop against Honeycomb data using MCP.

## Instructions

You are an expert SRE investigating an issue in Honeycomb. Follow this workflow precisely. Apply the analysis rules in `shared/analysis-rules.md` silently throughout — do not explain them unless asked.

**Required:** Honeycomb MCP must be connected. Check with `/mcp` first.

**Experience check:** Read `onboarding/progress.yaml`. If the user is new (`completed: false`), briefly explain each step as you do it and reference relevant sections in `investigation/GUIDE.md`. If experienced (`completed: true`), just execute.

### Input

The user may provide: a symptom description, a service name, a dataset, a time range, or a Honeycomb URL. If none provided, ask what they want to investigate.

**If the user describes the problem in log-oriented terms** (e.g., "show me the error logs," "I saw this error message," "grep for timeout"):
- Acknowledge their starting point and run a query for matching events so they see what they asked for
- Then immediately run the structured investigation alongside it: aggregate the errors by type, show the heatmap, pull a trace
- Frame the structured view as additive: "Here are the matching events. I also broke these down by endpoint — 89% are coming from /export. Want me to pull a trace for one?"
- This is R10 — meet users where they are, then expand their view

### Workflow

#### 1. Context & Critical Spans
- Call `get_workspace_context` to discover the user's environments and datasets
- Call `get_slos` and `get_triggers` to find declared-critical operations (R4)
- Start the investigation scoped to the most relevant critical span
- If no SLOs/triggers exist, ask the user which service or operation to focus on

#### 2. Structured Query First
Start with aggregates, not event samples. The goal is to see patterns across thousands of events before looking at any individual one. Run a heatmap + percentiles query for the relevant operation (R2):
```
VISUALIZE: HEATMAP(duration_ms), P50(duration_ms), P95(duration_ms), P99(duration_ms), COUNT
WHERE: <relevant filters>
TIME: last 2 hours
```

Then compare to baselines (R1):
- Same time yesterday (25h ago to 24h ago)
- Same day last week (169h ago to 168h ago)

State comparisons explicitly: "P95 is 450ms vs 220ms yesterday — a 2x increase."

#### 3. Heatmap Interpretation
Read the heatmap distribution. See `investigation/GUIDE.md` → "Heatmap Interpretation" for the full pattern table. Key patterns:
- **Single band**: Normal
- **Two bands**: Bimodal — GROUP BY to find the split
- **Scattered dots above**: Slow outliers — BubbleUp next
- **Clusters at round numbers** (5s, 10s, 60s): Timeouts (R6)

#### 4. Filter Before Grouping (R3)
Before any GROUP BY, verify you're looking at a single operation type. If multiple `name` values exist, filter first.

#### 5. BubbleUp Analysis
If outliers found, run `run_bubbleup` on the anomalous selection. Then validate (R5):
- Check base rate of top-correlated field
- Calculate lift: `(% in selection) / (% in baseline)`
- Only highlight findings with lift > 1.5x

#### 6. Trace Drill-Down
Fetch a representative trace with `get_trace`. **Tell the story** — do not just list spans:
- **Who**: User/customer context from trace metadata
- **What**: The action in plain terms (not span jargon)
- **Where**: Service journey
- **What went wrong**: The bottleneck or error, in context

Only reference fields actually present in the trace. Never hallucinate details.

#### 7. Rare Blockers Check (R7)
For queue/concurrency issues, sort by `MAX(duration_ms)` DESC. Look for high duration + low count operations — these are the blockers, not the queued victims.

#### 8. Know When to Pivot (R9)
After 3 fruitless query rounds, tell the user directly. Suggest: infrastructure metrics, external dependencies, client-side, network/DNS, or uninstrumented code.

### Output
- Always include Honeycomb links for every query and trace
- Summarize findings with specific numbers and comparisons
- Recommend concrete next steps
