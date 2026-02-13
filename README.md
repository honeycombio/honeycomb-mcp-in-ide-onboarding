# Honeycomb MCP Skills Pack

Learn to investigate your production behavior from inside your code editor using AI + Honeycomb. Instead of learning query syntax and UI navigation, you'll **orchestrate investigations** by directing an AI assistant that queries Honeycomb on your behalf—using your team's real telemetry data.

## What to Expect

This is a different way to learn observability:

- **You collaborate with an AI assistant** in your IDE or CLI, not in a web UI
- **You describe what you want to know** in plain language—the AI builds and runs Honeycomb queries for you
- **You see results + Honeycomb UI links** for visual confirmation and shareable artifacts
- **Your learning is driven by your actual data**—not abstract examples or sample datasets

**New to using AI for technical work?** That's okay! This skills pack guides you through it. Think of the AI as a knowledgeable teammate who knows Honeycomb deeply and can run queries for you. You just need to tell it what you're trying to figure out.

## Quick Start

1. **Choose an AI assistant** — We recommend [Claude Code](https://claude.ai/claude-code) (easiest setup)
2. **Clone this repo:** `git clone [url]` or download and extract the files
3. **Configure your AI tool** — See "Setup by AI Tool" below for instructions specific to your tool
4. **Start prompting:** `"Help me get started with Honeycomb"`

Your AI assistant will:
- Check if the Honeycomb MCP is connected and guide you through setup if needed
- Ask what you're trying to do: debugging, exploring, or checking reliability
- Guide you through real investigations using your production data
- Track your progress so it never repeats explanations

---

## Setup by AI Tool

### Claude Code

Claude Code automatically reads `CLAUDE.md` files in your working directory.

```bash
# Option A: Copy to your project root
cp -r honeycomb-skills/ ~/my-project/
cd ~/my-project
claude  # Start Claude Code from this directory

# Option B: Copy to home directory for global access
cp -r honeycomb-skills/ ~/honeycomb-skills/
cd ~/honeycomb-skills
claude
```

That's it—Claude Code will automatically pick up the `CLAUDE.md` instructions.

### Claude Desktop (via Projects)

The easiest setup—upload the folder to a Project and the instructions persist across all conversations.

1. Create a new Project in Claude Desktop (or use an existing one)
2. Upload all files from this folder to the Project's knowledge base
3. Add the Honeycomb MCP — see the [Honeycomb MCP configuration guide](https://docs.honeycomb.io/integrations/mcp/configuration-guide/) for Claude Desktop setup
4. Restart Claude Desktop and authenticate via the OAuth prompt
5. Start a conversation in the Project and ask: "Help me get started with Honeycomb"

Claude will automatically have access to all the guides and will follow the instructions in `CLAUDE.md`.

### ChatGPT Desktop (via Projects)

1. Create a new Project in ChatGPT
2. Upload all files from this folder to the Project
3. Add the Honeycomb MCP via Settings → MCPs (see MCP config below)
4. Start a conversation in the Project and ask: "Help me get started with Honeycomb"

**Alternative:** Add the contents of `CLAUDE.md` to ChatGPT's custom instructions (Settings → Personalization → Custom Instructions) for access outside of Projects.

### Cursor

Copy the contents of `CLAUDE.md` into Cursor's rules file:

```bash
cp honeycomb-skills/CLAUDE.md ~/my-project/.cursorrules
```

### Windsurf

```bash
cp honeycomb-skills/CLAUDE.md ~/my-project/.windsurfrules
```

### GitHub Copilot

```bash
mkdir -p ~/my-project/.github
cp honeycomb-skills/CLAUDE.md ~/my-project/.github/copilot-instructions.md
```

### Other AI Tools

Point your AI at `onboarding/GUIDE.md` and ask it to follow the instructions.

---

## Honeycomb MCP Setup

See the [Honeycomb MCP configuration guide](https://docs.honeycomb.io/integrations/mcp/configuration-guide/) for full setup instructions across all supported tools.

### Claude Code

**Easiest setup—the onboarding guide will do this for you automatically!**

When you start onboarding, Claude will detect if the MCP is missing and set it up for you. Just start with "Help me get started with Honeycomb" and follow the prompts.

**Manual setup (if preferred):**
```bash
claude mcp add honeycomb --transport http https://mcp.honeycomb.io/mcp
# Exit and restart Claude Code
# Run: /mcp to authenticate
```

For EU region, use `https://mcp.eu1.honeycomb.io/mcp` instead.

### Other Tools

See the [configuration guide](https://docs.honeycomb.io/integrations/mcp/configuration-guide/) for tool-specific instructions including Claude Desktop, VS Code, Amazon Q Developer, and more.

Authentication happens via OAuth when you first connect — no API keys needed.

---

## Verify Your Setup (Smoke Test)

After setting up the MCP, verify everything works before starting the learning paths. Copy and paste these prompts to your AI assistant:

### ✅ Step 1: Confirm MCP Connection

```
Confirm the Honeycomb MCP is connected and authenticated. List the Honeycomb MCP tools you have access to (names only). If not connected, tell me exactly what to do next.
```

**What should happen:** The AI should list MCP tools like `get_workspace_context`, `run_query`, `get_trace`, etc.

**If it fails:** The AI will guide you through setup or authentication. Follow its instructions, then try this prompt again.

---

### ✅ Step 2: Verify Data Access

```
Using Honeycomb MCP, verify you can access my workspace data. Identify the environment and dataset you will use (ask me at most 2 questions if needed). Then run the smallest safe query you can over the last 60 minutes that proves data is present. Show: time window, filters, and a Honeycomb link to the result.
```

**What should happen:**
- The AI runs a simple COUNT query on your data
- You see a number > 0 (proving data exists)
- You get a Honeycomb link you can click to see the query in the UI

**If it fails:**
- **"No data found"** → Check your time window (try last 24 hours instead)
- **"Authentication error"** → Run `/mcp` to re-authenticate
- **"Permission denied"** → Check your Honeycomb account permissions (see [PREREQUISITES.md](PREREQUISITES.md))

---

### 🎉 You're Ready!

If both prompts succeeded, you're all set. Now start your learning journey:

```
Help me get started with Honeycomb
```

The AI will ask what you're trying to do and guide you from there.

---


## How It Works

This skills pack works differently than traditional tutorials:

**Traditional approach:** Learn the query language → Learn the UI → Try it on your data → Debug production issues

**This approach:** Start with what you're trying to do → AI runs queries for you → Learn by seeing real results → Build understanding through practice

### Three Ways to Learn (Pick Your Path)

When you start, the AI will ask what you're trying to do. Choose based on your current need:

#### 🐛 **"I'm debugging something"**
You have a production issue to investigate right now. Learn Honeycomb by walking through a real debugging workflow:
- Finding anomalies in your telemetry
- Using BubbleUp to correlate patterns
- Drilling into traces to find bottlenecks
- **You'll learn:** Traces, spans, heatmaps, queries, BubbleUp

#### 🗺️ **"I'm just exploring"**
You want to understand what telemetry exists and build a mental model of your services:
- Discovering what's instrumented
- Understanding your data structure
- Establishing baseline metrics
- **You'll learn:** Datasets, queries, fields, percentiles, data model

#### 🎯 **"I'm checking reliability"**
You want to understand your service's health and what "good" looks like:
- Reviewing SLOs and error budgets
- Understanding burn rates
- Investigating at-risk services
- **You'll learn:** SLOs, SLIs, error budgets, burn rates, compliance

**All paths are equal**—start with whichever matches what you're trying to do right now. You can come back and try other paths later.

### For New vs. Experienced Users

**New to Honeycomb?** The AI teaches concepts as you encounter them, using your real data as examples. It tracks what you've learned so it never repeats explanations.

**Already experienced?** Mark yourself as `completed: true` in `onboarding/progress.yaml` and the AI will stay concise and out of the way—just helping when you need it.

## What's Included

```
honeycomb-skills/
├── CLAUDE.md                    # AI instructions (copy to your tool's config)
├── onboarding/
│   ├── GUIDE.md                 # Interactive onboarding—learns your experience level
│   ├── setup-mcp.md             # Automated MCP setup guide (OAuth-based)
│   ├── progress.yaml            # Tracks what you've learned (auto-updated)
│   └── my-context.yaml          # Your role and preferences (optional)
├── investigation/
│   └── GUIDE.md                 # Expert debugging heuristics and techniques
├── slo-basics/
│   └── GUIDE.md                 # Understanding SLOs, SLIs, and error budgets
└── shared/
    └── honeycomb-concepts.md    # Reference glossary of Honeycomb terminology
```

## Customization

Edit `onboarding/my-context.yaml` to set your preferences:

```yaml
role: product_engineer          # Your role (product_engineer, sre, data, manager)
experience_with_observability: none  # none, some, experienced
primary_services: [checkout, payments]  # Services you work on most
explanation_level: detailed     # detailed, normal, minimal
```

## Requirements

- A Honeycomb account with data flowing
- Any AI coding assistant that supports custom instructions
- The [Honeycomb MCP](https://github.com/honeycombio/honeycomb-mcp) connected to your AI tool

## What Success Looks Like

You'll know you've successfully learned a path when you can:

**Debugging path:**
- Describe a production symptom and narrow it to a root cause hypothesis
- Use structured queries (aggregates, breakdowns) before looking at individual events
- Pull representative traces and identify bottlenecks
- Share Honeycomb links as evidence in incident channels

**Exploring path:**
- Identify which datasets contain your service's data
- Know the key operations/endpoints and their baseline metrics
- Run queries to slice data by the dimensions that matter
- Create saved views you can reference later

**Reliability path:**
- Explain what your SLOs measure and why those targets were chosen
- Interpret error budgets and burn rates
- Investigate when an SLO is at risk
- Recommend alerting thresholds based on actual reliability data

**The ultimate goal:** You should feel comfortable using your AI assistant to investigate production issues on your own—orchestrating queries, interpreting results, and building hypotheses without needing to remember query syntax or UI navigation.

## Feedback

Found something confusing? Want to add a skill? Open an issue or submit a PR.
