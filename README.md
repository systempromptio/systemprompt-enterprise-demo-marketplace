# systemprompt-enterprise-demo-marketplace

A Claude Code plugin marketplace demonstrating **per-user token injection** via environment variables. Install plugins from this public marketplace, set your personal token and platform URL once, and every hook call authenticates as you.

## Quick Start

### 1. Get your credentials

Open your admin dashboard (e.g. `https://your-instance.systemprompt.io/admin/`) and click the **download icon** in the header. The install widget shows your:

- **SYSTEMPROMPT_URL** — your tenant's platform URL
- **SYSTEMPROMPT_TOKEN** — your personal JWT token (click the eye icon to reveal, then copy)

### 2. Configure Claude Code / Cowork

Add the credentials to `~/.claude/settings.json`:

```json
{
  "env": {
    "SYSTEMPROMPT_URL": "https://your-instance.systemprompt.io",
    "SYSTEMPROMPT_TOKEN": "your-jwt-here"
  }
}
```

**On Windows:** The file is at `%USERPROFILE%\.claude\settings.json`.

### 3. Install a plugin

**Via Claude Code CLI:**
```bash
claude plugin marketplace add systempromptio/systemprompt-enterprise-demo-marketplace
```

Then install either plugin:
- `enterprise-governance` — security review and compliance checks
- `developer-tools` — code review and standup summaries

**Via Cowork:**
Copy the GitHub URL from the install widget and paste it into Cowork's marketplace add dialog.

### 4. Verify

Use Claude normally — any tool use triggers hooks automatically. Check your dashboard for incoming events:

```bash
systemprompt analytics overview
```

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

### Session validation

On `SessionStart`, the plugin validates that both environment variables are set. If either is missing, it returns an error message telling you to configure them:

```json
{
  "SessionStart": [{
    "hooks": [{
      "type": "command",
      "command": "if [ -z \"${SYSTEMPROMPT_URL}\" ] || [ -z \"${SYSTEMPROMPT_TOKEN}\" ]; then echo '{\"error\": \"Missing SYSTEMPROMPT_URL or SYSTEMPROMPT_TOKEN\"}'; exit 1; fi && echo '{\"result\": \"ok\"}'",
      "async": false
    }]
  }]
}
```

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
| `SessionStart` | When Claude starts | Validates env vars are configured |
| `PreToolUse` | Before any tool executes | Tool name, input, session context |
| `PostToolUse` | After a tool completes | Tool name, output, duration, success/failure |
| `Stop` | When the agent stops | Reason, final message |
| `Notification` | On notification events | Notification content |

## Testing

### Test failure mode

Remove `SYSTEMPROMPT_TOKEN` from settings.json. Hooks still fire but the server returns 401. Claude continues working — hooks fail silently.

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
