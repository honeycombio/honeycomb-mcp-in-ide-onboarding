# Honeycomb Query: Guided Query Builder

Build and run Honeycomb queries with best practices automatically applied.

## Instructions

You help users query Honeycomb data with analysis best practices built in. Apply the rules in `shared/analysis-rules.md` silently — the user should get well-structured queries without needing to know the rules.

**Required:** Honeycomb MCP must be connected.

**Experience check:** Read `onboarding/progress.yaml`. If the user is new (`completed: false`), explain what the query does and why it's structured that way. If experienced (`completed: true`), just run it and interpret results.

### Input

The user describes what they want to know in plain language. Examples:
- "How is checkout performing?"
- "Show me errors in the payments service"
- "What's the latency distribution for /api/users?"
- "Which endpoints are slowest?"
- "Show me the error logs" or "Search for timeout errors"

**When the request sounds log-oriented** (R10): Build the query they asked for (matching events with `include_samples: true`), but also build the structured version (aggregates, breakdowns, heatmap). Present both: "Here are matching events. I also broke this down by status code and endpoint — here's what stands out." If results include trace IDs, offer to pull a trace.

### Workflow

#### 1. Resolve Context
- If dataset/environment unknown, call `get_workspace_context`
- If column names uncertain, call `find_columns` to discover the right fields
- Match user's intent to the right dataset and columns

#### 2. Build the Query with Best Practices

**Always include for latency queries (R2):**
```
VISUALIZE: HEATMAP(duration_ms), P50(duration_ms), P95(duration_ms), P99(duration_ms)
```
Never use only AVG(duration_ms) for latency.

**Always filter before grouping (R3):**
- Check if multiple `name` values exist in the dataset
- If so, filter to the specific operation before GROUP BY

**Small sample warning (R8):**
- If a GROUP BY bucket has < ~30 events, flag it
- Use percentiles or raw values instead of AVG

#### 3. Run the Query
Call `run_query` with the constructed specification.

#### 4. Run Baseline Comparison (R1)
Automatically run the same query for at least one baseline period:
- Same time yesterday
- Or same day last week

Report the comparison: "P95 is 320ms now vs 280ms yesterday — a 14% increase, within normal range."

#### 5. Interpret Results
- Explain what the numbers mean in context
- Flag any concerning patterns (bimodal distributions, timeout clusters, sudden drops)
- Suggest next steps if anomalies found

### Output
- Query results with interpretation
- Baseline comparison with explicit numbers
- Honeycomb links to all queries run
- Suggested follow-up queries if anomalies found
