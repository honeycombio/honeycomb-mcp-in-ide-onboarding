# SLO Check: Honeycomb Reliability Review

Check the health of SLOs and triggers for a specific service, with actionable next steps.

## Instructions

You are reviewing service reliability using Honeycomb SLOs. Follow this workflow. Apply the analysis rules in `shared/analysis-rules.md` silently throughout.

**Required:** Honeycomb MCP must be connected.

**Experience check:** Read `onboarding/progress.yaml`. If the user is new (`completed: false`), explain SLO concepts as you encounter them — reference `slo-basics/GUIDE.md` for definitions of error budgets, burn rates, and compliance. If experienced (`completed: true`), just show the data.

### Input

**There's no MCP tool that lists all SLOs in an environment**, so this can't run as a blind environment-wide sweep. The user should provide a service or dataset name. If they don't, ask: "Which service or dataset do you want to check reliability for?" — don't try to guess by scanning everything.

### Workflow

#### 1. Discover What Exists
- Call `get_workspace_context` if environment unknown
- Call `get_dataset_columns` for the named dataset(s) and look for `sli.*`-prefixed derived columns
- Call `get_triggers` for the environment — SLO burn alerts often appear here
- **Send a direct link to the Honeycomb SLOs page** for this environment and ask the user to confirm what's actually configured — this is the only reliable source of the SLO's name, target, and time window

#### 2. Compute Current Status
For any `sli.*` column found:
- Run `run_query` against it to compute compliance over the relevant window
- Ask the user for the SLO's target (from the UI) to judge whether that compliance is healthy, or use it directly if they tell you
- Compare to baselines: yesterday and last week (R1)

**Red flags (act now):** compliance well below target, or the user reports budget remaining < 10% / burn rate > 10x / status Triggered from the UI.

**Yellow flags (investigate soon):** compliance trending down, or budget remaining 10-30% / burn rate 1-3x.

**Green (healthy):** compliance comfortably above target, or budget remaining > 50% / burn rate < 1x.

#### 3. For Any At-Risk SLO
- Query for failing events matching the SLI filter
- Run BubbleUp on failures if count is significant — validate with base rates (R5)
- Compare failure rate to baselines: yesterday and last week (R1)

#### 4. Trigger Status
For each trigger, note:
- Is it currently firing?
- What condition triggers it?
- Which SLO (if any) does it relate to?

If the SLO is at-risk and no trigger references it, ask whether to attach a burn alert (`create_trigger` scoped to the `sli.*` column with `baseline_details`, plus `create_recipient`/`list_recipients` for a destination). Don't create one unprompted.

#### 5. If Nothing Exists
If there's no `sli.*` column and the user confirms nothing's in the UI, run a proxy reliability baseline (see `slo-basics/GUIDE.md` → "If no SLOs exist — proxy baseline") and **ask** whether they want to create one — don't call `create_slo` unprompted.

### Output
- Summary: SLO name (from the user/UI), compliance you computed, trigger status
- For at-risk SLOs: root cause analysis with specific numbers
- Recommendations: what to investigate, whether to delay deploys
- Include Honeycomb links to the SLOs page, every trigger, and every query
