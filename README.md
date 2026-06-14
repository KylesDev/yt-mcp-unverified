# yt-mcp

[![Tests](https://github.com/velesnitski/yt-mcp/actions/workflows/test.yml/badge.svg)](https://github.com/velesnitski/yt-mcp/actions/workflows/test.yml)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Release](https://img.shields.io/github/v/release/velesnitski/yt-mcp)](https://github.com/velesnitski/yt-mcp/releases)
[![Tools](https://img.shields.io/badge/tools-66-purple.svg)](#available-tools-66)
[![Docker](https://img.shields.io/badge/docker-velesnitski%2Fyt--mcp-blue?logo=docker)](https://hub.docker.com/r/velesnitski/yt-mcp)
[![Stars](https://img.shields.io/github/stars/velesnitski/yt-mcp?style=social)](https://github.com/velesnitski/yt-mcp/stargazers)

YouTrack MCP server for [Claude Code](https://claude.com/claude-code), [GitHub Copilot](https://github.com/features/copilot), [Cursor](https://cursor.com), [JetBrains IDEs](https://www.jetbrains.com/), and any MCP-compatible client.

## Quick start

### 1. Get a YouTrack permanent token

1. Open YouTrack → **Profile** → **Account Security** → **Tokens** → **New token**
2. Set scope: `YouTrack`
3. Grant permissions: **Read Issue**, **Write Issue**, **Read Project** (or just use admin token for full access)
4. Copy the token — it starts with `perm:`

### 2. Install in Claude Code

**Option A — via CLI** (recommended):

> **Prerequisite:** [uv](https://docs.astral.sh/uv/) is required. Install with: `curl -LsSf https://astral.sh/uv/install.sh | sh`

```bash
claude mcp add youtrack \
  -e YOUTRACK_URL=https://your-instance.youtrack.cloud \
  -e YOUTRACK_TOKEN=perm:your-token-here \
  -- uvx --from git+https://github.com/velesnitski/yt-mcp yt-mcp
```

If your YouTrack instance uses a self-signed or privately-issued certificate and you want to skip TLS verification, add:

```bash
  -e YOUTRACK_VERIFY_SSL=false
```

**Option B — manually edit settings:**

Edit `~/.claude.json` (global) or `.claude/settings.json` in your project root (per-project):

```json
{
  "mcpServers": {
    "youtrack": {
      "command": "uvx",
      "args": ["--from", "git+https://github.com/velesnitski/yt-mcp", "yt-mcp"],
      "env": {
        "YOUTRACK_URL": "https://your-instance.youtrack.cloud",
        "YOUTRACK_TOKEN": "perm:your-token-here",
        "YOUTRACK_VERIFY_SSL": "false"
      }
    }
  }
}
```

> **Troubleshooting: MCP server not found**
>
> If Claude Code can't find `uvx` at startup (e.g., `/mcp` doesn't show the server), use the full path instead:
>
> ```json
> "command": "/full/path/to/uvx"
> ```
>
> Find the full path with `which uvx` (typically `~/.local/bin/uvx` or `/opt/homebrew/bin/uvx`).

### 3. Restart Claude Code

```bash
claude
```

You should see `youtrack` listed when Claude starts. Try asking: *"List my YouTrack projects"*

## Multi-instance setup

Connect multiple YouTrack instances to a single MCP server. Each tool gets an optional `instance` parameter — the LLM picks the right instance from context, or auto-detects it when you paste a YouTrack URL.

### Configuration

Set `YOUTRACK_INSTANCES` to a comma-separated list of instance names, then provide `YOUTRACK_{NAME}_URL` and `YOUTRACK_{NAME}_TOKEN` for each:

```bash
YOUTRACK_INSTANCES=main,second
YOUTRACK_MAIN_URL=https://main.youtrack.cloud
YOUTRACK_MAIN_TOKEN=perm:xxx
YOUTRACK_SECOND_URL=https://second.youtrack.cloud
YOUTRACK_SECOND_TOKEN=perm:yyy
```

Adding more instances follows the same pattern:

```bash
YOUTRACK_INSTANCES=main,second,third,fourth
YOUTRACK_MAIN_URL=https://main.youtrack.cloud
YOUTRACK_MAIN_TOKEN=perm:xxx
YOUTRACK_SECOND_URL=https://second.youtrack.cloud
YOUTRACK_SECOND_TOKEN=perm:yyy
YOUTRACK_THIRD_URL=https://third.youtrack.cloud
YOUTRACK_THIRD_TOKEN=perm:zzz
YOUTRACK_FOURTH_URL=https://fourth.youtrack.cloud
YOUTRACK_FOURTH_TOKEN=perm:www
```

Instance names are arbitrary — use whatever makes sense: `prod,staging,dev`, `team1,team2`, etc. The name is uppercased to form the env var prefix (`dev` → `YOUTRACK_DEV_URL`).

### How it works

- **No `YOUTRACK_INSTANCES`** — single-instance mode, fully backward compatible. Uses `YOUTRACK_URL` / `YOUTRACK_TOKEN` as before.
- **First instance** falls back to unprefixed `YOUTRACK_URL` / `YOUTRACK_TOKEN` if its prefixed vars are not set.
- **URL auto-detection** — when you paste a YouTrack URL (e.g., `https://second.youtrack.cloud/issue/PROJ-123`), the server matches the domain to the right instance automatically.
- **Explicit `instance` parameter** — every tool accepts an optional `instance` param to target a specific instance.
- **Default** — if no instance is specified and no URL is provided, the first configured instance is used.
- **Global settings** — `YOUTRACK_READ_ONLY`, `DISABLED_TOOLS`, `YOUTRACK_MAX_BULK_RESULTS`, and `YOUTRACK_VERIFY_SSL` apply to all instances.

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `YOUTRACK_URL` | Yes | Your YouTrack instance URL (e.g., `https://company.youtrack.cloud`) |
| `YOUTRACK_TOKEN` | Yes | Permanent token (starts with `perm:`) |
| `YOUTRACK_INSTANCES` | No | Comma-separated instance names for multi-instance setup (e.g., `main,second`) |
| `YOUTRACK_READ_ONLY` | No | Set to `true` to disable all write operations |
| `DISABLED_TOOLS` | No | Comma-separated list of tools to disable (e.g., `delete_issue,bulk_update_execute`) |
| `YOUTRACK_MAX_BULK_RESULTS` | No | Maximum issues per bulk operation (default: `100`) |
| `YOUTRACK_ALLOW_HTTP` | No | Set to `1` to allow non-HTTPS URLs (not recommended) |
| `YOUTRACK_VERIFY_SSL` | No | Set to `false` to disable TLS certificate verification for YouTrack API requests |
| `YOUTRACK_OAUTH_URL` | No | Public HTTPS URL of this server — enables OAuth for claude.ai connectors |
| `YOUTRACK_ACCESS_CODE` | No | Access code for OAuth gate — users must enter this code to connect (requires `YOUTRACK_OAUTH_URL`) |
| `SENTRY_DSN` | No | Sentry DSN for error tracking — just set the env var, SDK is included |
| `YOUTRACK_LOG_FILE` | No | Error log path (default: `~/.yt-mcp/yt-mcp.log`) |
| `YOUTRACK_ANALYTICS_FILE` | No | Analytics log path (default: `~/.yt-mcp/analytics.log`) |
| `YOUTRACK_COMPACT` | No | Set to `1` to strip markdown from tool responses (~60% token savings) |

## Verify the server works

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"},"protocolVersion":"2024-11-05"}}' \
  | YOUTRACK_URL="https://your-instance.youtrack.cloud" \
    YOUTRACK_TOKEN="perm:your-token-here" \
    YOUTRACK_VERIFY_SSL="false" \
    uvx --from git+https://github.com/velesnitski/yt-mcp yt-mcp
```

## License

MIT
