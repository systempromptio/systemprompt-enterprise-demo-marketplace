# systemprompt-enterprise-demo-marketplace

A Claude Code plugin marketplace demonstrating **HTTP hooks with per-user token injection**. Install plugins from this public marketplace, set your personal token once, and every hook call authenticates as you.

## Quick Start

### Install via marketplace (Claude Code)

```bash
claude plugin marketplace add systemprompt/systemprompt-enterprise-demo-marketplace
```

Then install either plugin:
- `enterprise-governance` — security review and compliance checks
- `developer-tools` — code review and standup summaries

### Install via ZIP (Cowork)

1. Download or clone this repo
2. ZIP the plugin folder (e.g., `plugins/enterprise-governance/`)
3. In Cowork, go to Customize > Upload Plugin > drag the ZIP

## Token Setup

After installing a plugin, set your personal token in `~/.claude/settings.json`:

```json
{
  "env": {
    "SYSTEMPROMPT_TOKEN": "your-token-here"
  }
}
```

Get your token from [systemprompt.io](https://systemprompt.io).

**On Windows:** The file is at `%USERPROFILE%\.claude\settings.json`.

## How It Works

### The Problem

This marketplace is public — the same `hooks.json` is installed for every user. But each user needs their own identity token in hook HTTP headers.

### The Solution

Claude Code's HTTP hooks support environment variable interpolation in headers via `allowedEnvVars`:

```json
{
  "type": "http",
  "url": "https://app.systemprompt.io/api/public/hooks/post-tool-use",
  "headers": {
    "Authorization": "Bearer $SYSTEMPROMPT_TOKEN"
  },
  "allowedEnvVars": ["SYSTEMPROMPT_TOKEN"]
}
```

- `$SYSTEMPROMPT_TOKEN` in the header value is replaced with the user's env var at runtime
- `allowedEnvVars` explicitly permits this variable (unlisted vars resolve to empty strings)
- The token value lives in each user's local `~/.claude/settings.json`, never in the repo

### Security Properties

- No tokens in the repository (public repo is safe)
- Each user's token stays on their machine
- `allowedEnvVars` prevents accidental exfiltration of other env vars
- Server validates the Bearer token and identifies the user

## Hook Events

Both plugins track these events:

| Event | When It Fires | What's Sent |
|-------|--------------|-------------|
| `PreToolUse` | Before any tool executes | Tool name, input, session context |
| `PostToolUse` | After a tool completes | Tool name, output, duration, success/failure |
| `Stop` | When the agent stops | Reason, final message |
| `Notification` | On notification events | Notification content |

**Note:** `SessionStart` is not included because HTTP hooks are not supported for that event type (only `command` hooks work for `SessionStart`).

## Customizing

### Point at your own endpoint

Fork this repo and change the URLs in `hooks/hooks.json`:

```json
{
  "url": "https://your-server.com/hooks/post-tool-use"
}
```

### Add more hook events

Supported events for HTTP hooks: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Stop`, `Notification`, `SubagentStart`, `SubagentStop`, `PreCompact`, `UserPromptSubmit`, `PermissionRequest`, `TaskCompleted`, `TeammateIdle`, `ConfigChange`, `WorktreeCreate`, `WorktreeRemove`.

### Add a matcher

Filter hooks to specific tools:

```json
{
  "matcher": "Bash",
  "hooks": [{ "type": "http", "url": "...", ... }]
}
```

### Use multiple env vars

```json
{
  "headers": {
    "Authorization": "Bearer $SYSTEMPROMPT_TOKEN",
    "X-Team-ID": "$TEAM_ID"
  },
  "allowedEnvVars": ["SYSTEMPROMPT_TOKEN", "TEAM_ID"]
}
```

## Known Limitations

- **SessionStart**: HTTP hooks don't support `SessionStart`. Use a `command` hook with `curl` if you need session start tracking.
- **URL interpolation**: Environment variables only work in `headers`, not in the `url` field ([GitHub issue #31653](https://github.com/anthropics/claude-code/issues/31653)).
- **Cowork marketplace**: Adding custom marketplace URLs in Cowork's UI currently requires GitHub. Use ZIP upload as an alternative.

## Plugin Structure

```
plugins/enterprise-governance/
  .claude-plugin/plugin.json    # Plugin manifest
  hooks/hooks.json              # HTTP hooks with token injection
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
