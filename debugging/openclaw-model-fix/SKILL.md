---
name: openclaw-model-fix
description: Fix Unknown model errors in OpenClaw by adding missing model IDs to models.providers config
read_when:
  - OpenClaw reports Unknown model error or FailoverError
  - A model shows as missing in openclaw models list
  - A model provider updated their API model IDs
  - Logs show "model_not_found" reason in model-fallback/decision
---

# OpenClaw: Fix Unknown model error

When OpenClaw reports `Unknown model: provider/model-id`, the model ID is not in the pi-ai built-in catalog. Fix by adding it manually.

## Quick diagnosis

```bash
# Check model status
openclaw models list

# Check gateway logs for model errors
journalctl --user -u openclaw-gateway --no-pager -n 50 | grep -i "unknown model\|failover"
```

## Understanding the problem

OpenClaw 2026.3.31 ships with a built-in pi-ai catalog that may be outdated. For example, DeepSeek's built-in catalog only lists `deepseek-chat` and `deepseek-reasoner` (to be deprecated 2026/07/24), but the actual API now uses `deepseek-v4-flash` and `deepseek-v4-pro`.

The error flow: Gateway starts → resolves model ref `deepseek/deepseek-v4-flash` → not found in pi-ai catalog → `FailoverError: Unknown model` → agent fails before replying.

Other providers (OpenAI, Anthropic) may also have outdated catalogs for bleeding-edge models.

## Fix steps

### 1. Confirm the model ID exists with the provider API

```bash
curl -s https://api.deepseek.com/models \
  -H "Authorization: Bearer ${DEEPSEEK_API_KEY}" | jq .data[].id
```

### 2. Add custom models in models.providers

Edit `~/.openclaw/openclaw.json`. **IMPORTANT**: Use `python3 -c "import json; ..."` to edit — direct YAML/JSON editing risks invalidating the config (the Gateway validates schemas strictly on restart).

Example adding DeepSeek V4 models:

```bash
python3 << 'PYEOF'
import json

with open("/root/.openclaw/openclaw.json") as f:
    data = json.load(f)

if "models" not in data:
    data["models"] = {"mode": "merge", "providers": {}}
elif "providers" not in data["models"]:
    data["models"]["providers"] = {}

data["models"]["providers"]["deepseek"] = {
    "baseUrl": "https://api.deepseek.com",
    "api": "openai-completions",  # DeepSeek uses OpenAI-compatible API
    "models": [
        {
            "id": "deepseek-v4-pro",
            "name": "DeepSeek V4 Pro",
            "input": ["text"],
            "contextWindow": 131072,
            "maxTokens": 8192,
            "cost": {"input": 0.3, "output": 1.2}
        },
        {
            "id": "deepseek-v4-flash",
            "name": "DeepSeek V4 Flash",
            "input": ["text"],
            "contextWindow": 131072,
            "maxTokens": 8192,
            "cost": {"input": 0.3, "output": 1.2}
        }
    ]
}

# Also add to agents.defaults.models allowlist
data["agents"]["defaults"]["models"]["deepseek/deepseek-v4-pro"] = {}
data["agents"]["defaults"]["models"]["deepseek/deepseek-v4-flash"] = {}

# Set as primary if desired
data["agents"]["defaults"]["model"]["primary"] = "deepseek/deepseek-v4-pro"

with open("/root/.openclaw/openclaw.json", "w") as f:
    json.dump(data, f, indent=2)
PYEOF
```

For MiniMax (Anthropic-compatible API), the pattern is:
```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "minimax": {
        "baseUrl": "https://api.minimaxi.com/anthropic",
        "api": "anthropic-messages",
        "authHeader": true,
        "models": [
          { "id": "MiniMax-M2.7", "name": "MiniMax M2.7", ... }
        ]
      }
    }
  }
}
```

### 3. Set as default

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "deepseek/deepseek-v4-pro" },
      "models": {
        "deepseek/deepseek-v4-pro": {},
        "deepseek/deepseek-v4-flash": {}
      }
    }
  }
}
```

### 4. Restart gateway

```bash
systemctl --user restart openclaw-gateway
sleep 3
systemctl --user status openclaw-gateway
```

### 5. Verify

```bash
# Check model list with status
openclaw models list

# Check gateway logs for agent model resolution
journalctl --user -u openclaw-gateway --no-pager -n 10 | grep "agent model:"

# Ensure no Unknown model errors in recent logs
journalctl --user -u openclaw-gateway --no-pager --since "5 minutes ago" | grep -iE "unknown model|failover|model_not_found" || echo "✅ No model errors"
```

### 6. (Optional) Test end-to-end

Send a message through the gateway channel (QQ, WeCom, etc.) and check gateway logs:
```bash
journalctl --user -u openclaw-gateway --no-pager -n 20
```

## Identify the error source first

Not all "Unknown model" errors mean you need to edit `models.providers`. Some providers are **plugin-provided** and get their models dynamically. Run this diagnosis:

```bash
# 1. Check if it's a plugin-provided provider
ls /usr/lib/node_modules/openclaw/dist/extensions/ | grep -i "<provider>"

# 2. Check if the plugin registers the provider
cat /usr/lib/node_modules/openclaw/dist/extensions/<provider>/openclaw.plugin.json | python3 -m json.tool

# 3. Check the built-in model catalog (for plugin providers)
# For volcengine-plan, the models are in dist/models-BZGrbXXp.js
grep -oP '"id":"[^"]*"' /usr/lib/node_modules/openclaw/dist/models-BZGrbXXp.js
```

### Plugin-provided providers (no models.providers config needed)

Some providers ship as bundled plugins and populate their model catalog at runtime via the plugin API. You **do NOT** need to add them to `models.providers`:

| Provider | Plugin | Built-in Models |
|----------|--------|----------------|
| `volcengine` | `extensions/volcengine/` (`buildDoubaoProvider`) | doubao-seed-1-8-251228, kimi-k2-5-260127, glm-4-7-251222, deepseek-v3-2-251201 |
| `volcengine-plan` | `extensions/volcengine/` (`buildDoubaoCodingProvider`) | ark-code-latest, doubao-seed-code, glm-4.7, kimi-k2-thinking, kimi-k2.5 |
| `volcengine` normalizes: `bytedance`→`volcengine`, `doubao`→`volcengine` | | |
| `volcengine-plan` auth normalizes to `volcengine` (shares API key) | | |

The volcengine plugin registers both providers at `dist/extensions/volcengine/index.js`. The coding provider uses a different base URL: `https://ark.cn-beijing.volces.com/api/coding/v3`.

To verify these models are available to the running Gateway, check the logs for any model errors on startup.

### Manual models.providers — for providers not covered by plugins

Providers like DeepSeek and MiniMax are **not** plugin-provided and require manual entries in `models.providers`.

## OpenClaw Upgrade Procedure

```bash
# 1. Check current vs latest
openclaw version
npm view openclaw version

# 2. Upgrade (use official registry if tencent mirror fails)
npm install -g openclaw@latest --registry https://registry.npmjs.org

# 3. Install/update required plugins
openclaw plugins install @openclaw/qqbot  # QQ Bot now a separate npm plugin

# 4. Clean old extensions that conflict with new plugins
rm -rf ~/.openclaw/extensions/qqbot  # Old TS-source extension → conflicts with npm plugin

# 5. Restart
systemctl --user restart openclaw-gateway
```

### QQ Bot Plugin Migration (v2026.5.x+)

The QQ Bot adapter moved from built-in to an external npm plugin:

- **Old**: `~/.openclaw/extensions/qqbot/` (TypeScript source, not compiled)
- **New**: `openclaw plugins install @openclaw/qqbot` → `~/.openclaw/npm/node_modules/@openclaw/qqbot/dist/index.js`
- **After install**: delete the `.bak` file `openclaw plugins install` creates: `rm ~/.openclaw/openclaw.json.bak`

The old extension directory must be removed after installing the npm plugin, or the gateway will report:
```
plugin qqbot: installed plugin package requires compiled runtime output for TypeScript entry index.ts
```

**⚠️ After config cleanup**: always verify QQ Bot is still installed — `openclaw plugins list | grep qqbot`. If `~/.openclaw/npm/` was deleted during cleanup, reinstall it. See `openclaw-maintenance` skill for full cleanup verification.

### npm Registry Issues

Tencent npm mirror (`mirrors.tencentyun.com/npm`) may fail for certain dependencies (e.g., `pac-resolver@9.0.1`). Use the official registry:
```bash
npm install -g openclaw@latest --registry https://registry.npmjs.org
```

## Pitfalls

- **apiKeyEnvVar is NOT recognized** in models.providers — do not use it. OpenClaw's config schema rejects unknown keys on restart.
  - Error in logs: `Unrecognized key: \"apiKeyEnvVar\"` → Gateway won't start
- **models.mode must be \"merge\"** — otherwise it overwrites the entire provider catalog instead of merging
- API key is stored in `~/.openclaw/agents/main/agent/auth-profiles.json` separately, no need to duplicate in models.providers
- **Provider ID normalization**: `volcengine-plan` normalizes to `volcengine` for auth (they share the API key). `bytedance`/`doubao` normalize to `volcengine`.
- **Config validation is strict** — OpenClaw validates the entire JSON on startup. Even a running Gateway will detect config changes via file watch and fail if invalid. Use `python3 -c \"json.load(open(...))\"` to validate before restarting.
- **Built-in documentation**: `/usr/lib/node_modules/openclaw/docs/` — check `docs/providers/<provider>.md` for the official provider setup guide
- **DeepSeek doc**: `/usr/lib/node_modules/openclaw/docs/providers/deepseek.md` — lists models as `deepseek-chat`/`deepseek-reasoner` (outdated for V4)
- **DeepSeek official API**: https://api-docs.deepseek.com/ — confirms `deepseek-v4-pro` and `deepseek-v4-flash` as current models
- **DeepSeek model name differences**: Hermes uses `deepseek-v4-pro` as bare model name; OpenClaw uses `deepseek/deepseek-v4-pro` with provider prefix. They are different systems — set them independently.
- **Backup before editing**: Always back up `openclaw.json` before making changes: `cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%Y%m%d)`
- **Model config looks right but bot doesn't respond?** The issue may not be the model at all. Check QQ Bot plugin: `openclaw plugins list | grep qqbot`. If missing (`plugin not installed: qqbot`), install: `openclaw plugins install @openclaw/qqbot`. This is the #1 cause of "no response" after config cleanup or upgrades. See `openclaw-maintenance` skill for full diagnostic.
