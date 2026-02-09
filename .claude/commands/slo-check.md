# SLO Check: Honeycomb Reliability Review

Check the health of SLOs and triggers, with actionable next steps.

## Instructions

You are reviewing service reliability using Honeycomb SLOs. Follow this workflow. Apply the analysis rules in `shared/analysis-rules.md` silently throughout.

**Required:** Honeycomb MCP must be connected.

**Experience check:** Read `onboarding/progress.yaml`. If the user is new (`completed: false`), explain SLO concepts as you encounter them — reference `slo-basics/GUIDE.md` for definitions of error budgets, burn rates, and compliance. If experienced (`completed: true`), just show the data.

### Input

The user may provide: an environment name, a service name, or an SLO ID. If none, discover environments via `get_workspace_context` and check all SLOs in their default environment.

### Workflow

#### 1. Fetch SLOs and Triggers
- Call `get_workspace_context` if environment unknown
- Call `get_slos` for the environment
- Call `get_triggers` for the environment
- Present a summary table of all SLOs with status, compliance, and budget remaining

#### 2. Triage by Status

**Red flags (act now):**
- Budget remaining < 10%
- Burn rate > 10x
- Status: Triggered

**Yellow flags (investigate soon):**
- Budget remaining 10-30%
- Burn rate 1-3x
- Compliance dropping toward target

**Green (healthy):**
- Budget remaining > 50%
- Burn rate < 1x

#### 3. For Any At-Risk SLO
- Call `get_slos` with the specific `slo_id` for detailed view with graphs
- Show the Budget Burndown and Historical Compliance graphs
- Query for failing events matching the SLI filter
- Run BubbleUp on failures if count is significant — validate with base rates (R5)
- Compare failure rate to baselines: yesterday and last week (R1)

#### 4. Trigger Status
For each trigger, note:
- Is it currently firing?
- What condition triggers it?
- Which SLO (if any) does it relate to?

### Output
- Summary table: SLO name, target, compliance, budget remaining, status
- For at-risk SLOs: root cause analysis with specific numbers
- Recommendations: what to investigate, whether to delay deploys
- Include Honeycomb links to every SLO, trigger, and query
