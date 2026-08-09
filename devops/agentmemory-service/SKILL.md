---
name: agentmemory-service
description: "Install, configure, and manage agentmemory — persistent cross-agent memory server (BM25 + vector + knowledge graph). Integrates with Hermes, OpenClaw, Claude Code, and any MCP client."
version: 1.0.0
tags: [agentmemory, memory, iii-engine, mcp, hermes-plugin]
triggers:
  - agentmemory
  - agent memory
  - persistent memory server
  - cross-agent memory
  - iii-engine
---

# agentmemory Service

Persistent memory server for AI coding agents. Built on iii-engine, provides BM25 + vector + knowledge graph search with 95.2% retrieval accuracy. One server shared across all agents.

**Repo:** https://github.com/rohitg00/agentmemory
**Docs:** README.md in repo

## Architecture

```
agentmemory (Node.js worker)
  └── iii-engine (state engine, SQLite, port 49134)
      ├── REST API (port 3111)
      ├── WebSocket streams (port 3112)
      └── Viewer (port 3113-3115, fallback)
```

- 53 MCP tools, 125 REST endpoints
- State: file-based SQLite via iii-engine's StateModule
- Embedding: all-MiniLM-L6-v2 (local, free, no API key)

## Install

```bash
npm install -g @agentmemory/agentmemory
agentmemory          # start (foreground)
agentmemory demo     # seed sample data + verify recall
agentmemory stop     # stop
agentmemory doctor   # diagnostics
agentmemory remove   # full uninstall
```

## systemd Service

```ini
# /etc/systemd/system/agentmemory.service
[Unit]
Description=agentmemory - Persistent memory for AI coding agents
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/agentmemory
Environment=HOME=/root
Environment=PATH=/root/.nvm/versions/node/v24.14.1/bin:/root/.local/bin:/usr/local/bin:/usr/bin:/bin
WorkingDirectory=/root
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Use `command -v agentmemory` and adjust `ExecStart` if the binary is installed elsewhere.

## Hermes Integration

Two options (Option 2 is deeper):

### Option 1: MCP server only

Add to `~/.hermes/config.yaml`:
```yaml
mcp_servers:
  agentmemory:
    command: npx
    args: ["-y", "@agentmemory/mcp"]
```

### Option 2: Memory provider plugin (recommended)

```bash
# Print the current MCP/YAML guidance (the adapter does not copy the plugin yet)
agentmemory connect hermes

# Install/update the deeper provider plugin from an agentmemory source checkout
install -d ~/.hermes/plugins/agentmemory
cp -a <agentmemory-repo>/integrations/hermes/. ~/.hermes/plugins/agentmemory/
hermes plugins enable agentmemory

# Set provider in config.yaml
# memory:
#   provider: agentmemory
```

Plugin provides 6 lifecycle hooks:
- `prefetch()` — injects relevant memories before each LLM call
- `sync_turn()` — captures every conversation turn in background
- `on_session_end()` — marks sessions complete for summarization
- `on_pre_compress()` — re-injects context before compaction
- `on_memory_write()` — mirrors MEMORY.md writes to agentmemory
- `system_prompt_block()` — injects project profile at session start

### Environment variables

| Variable | Default | Description |
|---|---|---|
| `AGENTMEMORY_URL` | `http://localhost:3111` | Server URL |
| `AGENTMEMORY_SECRET` | (none) | Auth token |

Config file: `~/.agentmemory/.env` (auto-loaded by plugin).

## OpenClaw Integration

Add to MCP config:
```json
{
  "mcpServers": {
    "agentmemory": {
      "command": "npx",
      "args": ["-y", "@agentmemory/mcp"],
      "env": { "AGENTMEMORY_URL": "http://localhost:3111" }
    }
  }
}
```

For deeper slot integration: `plugins.slots.memory = "agentmemory"` in openclaw.json.

## Health Check

```bash
curl -s http://localhost:3111/agentmemory/health | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['status'], d['version'])"
```

## Pitfalls

- iii-engine must be running (agentmemory spawns it automatically, but if port 49134 is occupied by a stale process, kill it first)
- Viewer port falls back (3113 → 3114 → 3115) if occupied
- `npx` caches per-version — use `npm install -g` for reliable updates, not bare `npx`
- First run after install may timeout in foreground due to iii-engine bootstrap — systemd handles this gracefully with restart
