# Setting Up the Honeycomb MCP

This skill guides users through adding the Honeycomb MCP server to enable Honeycomb observability features in Claude Code.

---

## When to Use This Skill

This skill should be triggered when:
- User is starting onboarding (`started: false`) and MCP is not connected
- User tries to use Honeycomb commands but MCP is not configured
- User explicitly asks to set up or add the Honeycomb MCP

---

## Detection Logic

**Primary method:** Run the `/mcp` command to check MCP status.

The `/mcp` command will show:
- ✅ If the user is already authenticated to Honeycomb
- ⚙️ If the MCP is configured but needs authentication
- ❌ If the MCP is not configured at all

**Fallback method:** Look for these MCP tools:
- `mcp__honeycomb__*` or `mcp__dogfood-honeycomb__*` tools

If `/mcp` shows the MCP is not configured or tools are NOT available, guide the user through setup.

---

## Setup Flow

### Quick Check: Is MCP Already Configured?

If `/mcp` shows the Honeycomb MCP is configured but not authenticated:

> "I see the Honeycomb MCP is already set up! You just need to authenticate.
>
> Type `/mcp` and follow the authentication flow to connect to your Honeycomb account."

Then proceed to **Step 3: Verify Connection** below.

---

### Step 1: Add the MCP Server (If Not Configured)

Tell the user:

> "Let's connect the Honeycomb MCP server so I can query your observability data. I'll add it now..."

**Action:** Run this command via Bash:

```bash
claude mcp add honeycomb --transport http https://mcp.honeycomb.io/mcp
```

### Step 2: Exit and Restart

After the command succeeds, tell the user:

> "✅ Honeycomb MCP server added successfully!
>
> **Next steps:**
> 1. Exit this chat (Ctrl+C or Cmd+D)
> 2. Start a new Claude Code session
> 3. Type `/mcp` to authenticate with your Honeycomb account
>
> Once you authenticate, come back and say 'ready' to continue!"

**Update progress.yaml:**
```yaml
mcp_setup_pending: true
```

---

## After Authentication

When the user returns after running `/mcp`:

### Step 3: Verify Connection

**Action:** Try calling `get_workspace_context` or list available MCP tools.

If successful:

> "🎉 Connected to Honeycomb! I can now query your observability data.
>
> Let's explore what you have instrumented..."

**Update progress.yaml:**
```yaml
mcp_setup_pending: false
mcp_connected: true
```

Then proceed to the normal onboarding flow (Step 1: Discover What's Instrumented).

---

## Troubleshooting

If connection fails or MCP tools are still not available:

1. Check if the user ran `/mcp` to authenticate
2. Verify they exited and restarted Claude Code
3. Check for error messages during authentication
4. Suggest checking `~/.claude/mcp.json` for the honeycomb configuration

---

## OAuth Priority Note

The Honeycomb MCP prioritizes OAuth workflows for authentication, which is why the `/mcp` command triggers an authentication flow rather than requiring API keys in configuration.

This approach:
- Is more secure (no keys in config files)
- Provides better user experience (OAuth flow in browser)
- Supports team-based access control

---

## Integration with Onboarding

This setup step should occur **before** the "Discover What's Instrumented" step in the main onboarding guide.

The updated flow:
1. **Check MCP connection** (this skill)
2. Discover What's Instrumented
3. Choose Learning Path
4. Continue with selected path...
