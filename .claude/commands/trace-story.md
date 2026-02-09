# Trace Story: Narrate a Honeycomb Trace

Fetch a trace and tell its story in plain language, not span jargon.

## Instructions

You are explaining what happened during a specific request by reading its distributed trace. Your job is to make traces understandable — tell a story, not a span list.

**Required:** Honeycomb MCP must be connected.

**Experience check:** Read `onboarding/progress.yaml`. If the user is new (`completed: false`), introduce trace vocabulary after telling the story. If experienced (`completed: true`), lead with the interesting finding.

### Input

The user provides: a trace ID, a Honeycomb trace URL, or asks you to find an interesting trace from a query.

### If No Trace ID Provided
- Call `get_workspace_context` to discover environments/datasets
- Run a query to find an interesting trace:
```
VISUALIZE: HEATMAP(duration_ms), P95(duration_ms), COUNT
WHERE: <user's filters or service>
TIME: last 1 hour
```
- Pick a representative outlier trace from the results

### Workflow

#### 1. Fetch the Trace
Call `get_trace` with the trace ID and environment.

#### 2. Tell the Story
Narrate what happened in this order:

**Who** — Look at every field for user/customer context:
- User ID, email, account name
- Plan tier, team, organization
- Feature flags, A/B test groups
Example: "This was a request from a user on the Enterprise plan, team 'acme-corp'."

**What** — Describe the action using route/endpoint/operation names:
- Use plain terms: "They were exporting a CSV report" not "POST /api/v1/export was called"
- Include request parameters if available (payload size, filters, etc.)

**Where** — Narrate the service journey:
- "The request started at the API gateway, went to the export service, which queried the database and called S3"
- Note how many services were involved and the overall architecture

**How long** — Break down the timing:
- Total duration
- Where time was spent (as percentages of total)
- The critical path — which spans were on the longest chain

**What went wrong (or right)** — The interesting finding:
- The bottleneck span and why it's slow
- Any errors or retries
- Timeout signatures: round-number durations at 5s, 10s, 30s, 60s (R6)
- Parallel vs sequential execution patterns

#### 3. Technical Summary
After the narrative, provide:
- Total spans count
- Service count
- Critical path duration vs total duration
- Any error spans

### Rules
- **Never hallucinate details.** Only reference fields actually present in the trace
- **Use every piece of context available** — user IDs, feature flags, request params, etc.
- Include the Honeycomb trace link so the user can see the waterfall view
