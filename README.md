# systemprompt-enterprise-demo-marketplace

A Claude Code plugin marketplace demonstrating **per-user token injection** via environment variables. Install plugins from this public marketplace, set your personal token and platform URL once, and every hook call authenticates as you.

## Quick Start

### Install via marketplace (Claude Code)

```bash
claude plugin marketplace add systempromptio/systemprompt-enterprise-demo-marketplace
```

Then install either plugin:
- `enterprise-governance` — security review and compliance checks
- `developer-tools` — code review and standup summaries

### Install via ZIP (Cowork)

1. Download or clone this repo
2. ZIP the plugin folder (e.g., `plugins/enterprise-governance/`)
3. In Cowork, go to Customize > Upload Plugin > drag the ZIP

## Token Setup

After installing a plugin, set your platform URL and personal token in `~/.claude/settings.json`:

```json
{
  "env": {
    "SYSTEMPROMPT_URL": "https://your-instance.systemprompt.io",
    "SYSTEMPROMPT_TOKEN": "your-jwt-here"
  }
}
```

Get your token from your systemprompt.io admin dashboard.

**On Windows:** The file is at `%USERPROFILE%\.claude\settings.json`.

## How It Works

### The Problem

This marketplace is public — the same `hooks.json` is installed for every user. But each user needs:
1. Their own identity token in hook headers
2. Their own platform URL (subdomains are per-tenant)

### The Solution

Command hooks with `curl` use environment variables for both the URL and the Bearer token:

```json
{
  "type": "command",
  "command": "curl -s --max-time 10 -X POST \"${SYSTEMPROMPT_URL}/api/public/hooks/track?plugin_id=enterprise-governance\" -H \"Authorization: Bearer ${SYSTEMPROMPT_TOKEN}\" -H \"Content-Type: application/json\" -d \"$(cat)\" >/dev/null 2>&1 &\nexit 0",
  "async": true
}
```

- `${SYSTEMPROMPT_URL}` — your tenant's platform URL (set per-user in settings.json)
- `${SYSTEMPROMPT_TOKEN}` — your personal JWT (set per-user in settings.json)
- `$(cat)` — reads the hook event payload from stdin (Claude pipes it automatically)
- `>/dev/null 2>&1 &` — backgrounds curl so it never blocks Claude
- `async: true` — tells Claude not to wait for the hook to complete

### Why command hooks instead of HTTP hooks?

Claude Code's native HTTP hooks (`type: "http"`) support env var interpolation in headers via `allowedEnvVars`, but **not in the URL field** ([GitHub issue #31653](https://github.com/anthropics/claude-code/issues/31653)). Since the platform URL contains a per-tenant subdomain, we use command hooks with curl where env vars work everywhere.

### Security Properties

- No tokens or URLs in the repository (public repo is safe)
- Each user's credentials stay on their machine in `~/.claude/settings.json`
- Server validates the JWT and identifies the user
- Without a valid token, hooks fail silently — Claude keeps working normally

## Hook Events

Both plugins track these events:

| Event | When It Fires | What's Sent |
|-------|--------------|-------------|
| `PreToolUse` | Before any tool executes | Tool name, input, session context |
| `PostToolUse` | After a tool completes | Tool name, output, duration, success/failure |
| `Stop` | When the agent stops | Reason, final message |
| `Notification` | On notification events | Notification content |

## Testing

### 1. Set your env vars

Edit `~/.claude/settings.json` with your platform URL and JWT.

### 2. Install a plugin

Via marketplace add or ZIP upload (see Quick Start above).

### 3. Use Claude normally

Any tool use triggers `PreToolUse` and `PostToolUse` hooks automatically. The curl command posts to your platform with your JWT.

### 4. Verify on the platform

Check your dashboard for incoming events, or use the CLI:

```bash
systemprompt analytics overview
```

### 5. Test failure mode

Remove `SYSTEMPROMPT_TOKEN` from settings.json. Hooks still fire but the server returns 401. Claude continues working — hooks fail silently.

## Customizing

### Point at your own endpoint

Fork this repo and change the URL pattern in `hooks/hooks.json`. The endpoint receives a JSON POST with the hook event payload.

### Add more hook events

Supported events: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Stop`, `Notification`, `SubagentStart`, `SubagentStop`, `PreCompact`, `UserPromptSubmit`, `PermissionRequest`, `TaskCompleted`, `TeammateIdle`, `ConfigChange`, `WorktreeCreate`, `WorktreeRemove`.

### Add a matcher

Filter hooks to specific tools:

```json
{
  "matcher": "Bash",
  "hooks": [{ "type": "command", "command": "...", "async": true }]
}
```

## Plugin Structure

```
plugins/enterprise-governance/
  .claude-plugin/plugin.json    # Plugin manifest
  hooks/hooks.json              # Command hooks with curl + env vars
  skills/
    security-review/SKILL.md    # Auto-triggers on security review tasks
    compliance-check/SKILL.md   # Auto-triggers on compliance tasks
  agents/
    governance_agent.md         # Agent system prompt

plugins/developer-tools/
  .claude-plugin/plugin.json
  hooks/hooks.json
  skills/
    code-review/SKILL.md
    standup-summary/SKILL.md
```

## License

MIT
