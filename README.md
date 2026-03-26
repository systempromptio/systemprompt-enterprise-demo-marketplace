# Enterprise Demo — foodles.com

Demonstrates enterprise governance for Claude Code using HTTP hooks, MCP servers, and secret detection policies.

## What's Inside

- **2 skills**: `example-web-search` (allowed) and `use-dangerous-secret` (blocked by governance)
- **2 MCP servers**: `systemprompt` (admin tools) and `skill-manager` (user tools)
- **HTTP hooks**: PreToolUse governance hook that blocks plaintext secrets, plus tracking hooks for all events
- **No shell scripts** — all hooks use Claude Code's native `type: "http"` format

## Install

```bash
claude plugin marketplace add systempromptio/systemprompt-enterprise-demo-marketplace
```

## Setup

Add your token to Claude Code settings:

```json
{
  "env": {
    "SYSTEMPROMPT_TOKEN": "your-jwt-here"
  }
}
```

Get your token from the admin dashboard at [abc3dd581f80.systemprompt.io/admin](https://abc3dd581f80.systemprompt.io/admin/) — click the download icon in the header to reveal your JWT.

## Try It

### 1. Web Search (allowed)

Ask Claude to search the web. The governance hook evaluates the tool call, allows it, and tracks the event.

### 2. Dangerous Secret (blocked)

Ask Claude to use the dangerous secret skill. It will attempt to write a file containing `sk-ant-demo-FAKE12345678901234567890`. The PreToolUse governance hook detects the secret pattern and blocks the tool call.

## How It Works

### HTTP Hooks

All hooks use `type: "http"` with `allowedEnvVars` to resolve `$SYSTEMPROMPT_TOKEN` from your Claude settings at runtime:

```json
{
  "type": "http",
  "url": "https://abc3dd581f80.systemprompt.io/api/public/hooks/govern?plugin_id=enterprise-demo",
  "headers": {
    "Authorization": "Bearer $SYSTEMPROMPT_TOKEN"
  },
  "allowedEnvVars": ["SYSTEMPROMPT_TOKEN"],
  "timeout": 10
}
```

- **Governance** (`PreToolUse`): Synchronous. Returns `allow` or `deny` with a reason.
- **Tracking** (all other events): Async fire-and-forget analytics.

### MCP Servers

Both servers authenticate via OAuth — Claude Code handles the flow automatically on first use.

- **systemprompt**: Platform administration tools (admin scope)
- **skill-manager**: Skill and agent management tools (user scope)

### Governance Rules

The governance endpoint evaluates four rules:

1. **Secret detection** — scans tool inputs for API keys, tokens, passwords, connection strings
2. **Scope check** — enforces admin-only tool restrictions based on agent scope
3. **Tool blocklist** — blocks destructive operations (delete, drop, destroy) for non-admin scopes
4. **Rate limiting** — 60 tool calls per minute per session

## Plugin Structure

```
plugins/enterprise-demo/
  .claude-plugin/plugin.json
  .mcp.json
  hooks/hooks.json
  skills/
    example-web-search/SKILL.md
    use-dangerous-secret/SKILL.md
```

## License

MIT
