---
name: system-config-inventory
description: Maintain a centralized system-config reference that can be dumped on demand (e.g. "init" command). Covers multi-service credentials, file paths, ports, plugin lists — for Hermes, OpenClaw, and any other services on the server.
read_when:
  - User says "init" or asks for system configuration overview
  - Setting up a new server and need a standard config inventory template
  - Migrating config between agents (e.g. OpenClaw → Hermes)
  - Debugging credential sync issues between agents
---

# System Config Inventory

Maintain a centralized reference file so the agent can dump the full system configuration on demand (e.g. when user says "init").

## Architecture

The init response is built from two sources:

1. **Template file** (`references/init-response.md`) — static skeleton with sections for each service
2. **Memory entries** — dynamic values that may change (ports, running status, current agent versions)

When user says "init":
1. Load the template from `references/init-response.md`
2. Cross-reference with memory for any updated values
3. Return the complete config dump

## Init Response Content

The full response should cover:

### Hermes Agent
- Version
- Active providers/models
- Dashboard URL + auth
- Gateway status (which platform channels are connected)

### OpenClaw (clawbot)
- Version, config path
- **QQ Bot**: appId, clientSecret, imageServerBaseUrl, chatId, plugin name + install command
- **企业微信 (WeCom)**: corpId, corpSecret, agentId, encodingAesKey, plugin name + install command
- **Gateway**: port, mode (local/lan), auth_token
- **Plugin list**: installed npm packages + their purpose
- **Workspace info**: agent bindings, shared paths

### Network Services
- nginx ports (HTTP/HTTPS)
- NoVNC/remote desktop URL + auth
- Edge CDP endpoint
- Any web applications (dashboard, games, etc.)

### File Paths
- Config files for each service
- Skill directories
- Memory/workspace locations

### OpenClaw Managed Skills
- List of installed skills (for reference if migrating to Hermes)

## Creating the Template

```markdown
# ⚙️ 系统配置总览

## 🤖 Hermes Agent
- 版本: x.x.x
- Dashboard: http://hostname/hermes/ (auth)
- 网关: running/stoped

## 🔌 OpenClaw
### QQ Bot
- appId: `xxx`
- clientSecret: `xxx`
- ...

### 企业微信
- corpId: `xxx`
- ...
```

## Pitfalls

- **Don't expose raw API keys/secrets** unless user explicitly asks. Use truncated forms like `tvly-d...NiMx`.
- **Keep the template in `~/.hermes/skills/devops/system-config-inventory/references/`**, not in `~/.hermes/` directly. Skills backup with git; loose files don't.
- **When migrating from OpenClaw → Hermes**, the init response format should mirror what OpenClaw returned, so the user's muscle memory ("发init就返回完整配置") works identically.
- **Template gets stale** — update it when you change a service port, add/remove a plugin, or rotate credentials. Memory entries serve as the "live" overlay.
- **User hates contextless extra work** — if they ask about init/config, just dump the answer. Don't also check system status, restart services, or offer unsolicited improvements unless asked.
