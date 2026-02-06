# Honeycomb Skills Pack

A collection of AI-assisted guides that help engineers learn Honeycomb through their actual production data. New engineers get guided onboarding; experienced users get concise, out-of-the-way assistance.

## Quick Start

1. Set up the Honeycomb MCP (see below)
2. Copy this folder and configure your AI tool
3. Ask: **"Help me get started with Honeycomb"**

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

## How It Works

**For new engineers:** The AI guides you through Honeycomb step-by-step, using your actual production data. It tracks which concepts you've learned so it never repeats explanations.

**For experienced users:** Once onboarding is complete, the AI stays concise and out of the way—just helping when you need it.

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
- The [Honeycomb MCP](https://github.com/honeycombio/honeycomb-mcp) connected to your AI tool
- Any AI coding assistant that supports custom instructions

## Learning Paths

The onboarding adapts to what you're trying to do:

- **"I'm debugging something"** → Learn traces by investigating a real issue
- **"Just exploring"** → Understand your data model and what's instrumented
- **"Checking reliability"** → Learn SLOs and how to monitor service health

## Feedback

Found something confusing? Want to add a skill? Open an issue or submit a PR.
