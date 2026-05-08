# QQ Bot Plugin Verification

## Check if installed

```bash
openclaw plugins list | grep qqbot
# Expected: │ @openclaw/qqbot │ qqbot │ openclaw │ enabled │ stock:qqbot/index.js │
```

Or check npm directory:
```bash
ls ~/.openclaw/npm/node_modules/@openclaw/qqbot/
```

## Install

```bash
openclaw plugins install @openclaw/qqbot
# Creates backup: ~/.openclaw/openclaw.json.bak — delete it
rm ~/.openclaw/openclaw.json.bak
```

## Healthy startup log pattern

```
[qqbot] [qqbot:default] Starting gateway — appId=102870423, enabled=true
[qqbot] [default] ✅ Access token obtained successfully
[qqbot] [default] Connecting to wss://api.sgroup.qq.com/websocket
[qqbot] [default] WebSocket connected
[qqbot] [qqbot:default] Gateway ready (or Gateway resumed)
[gateway] http server listening (3 plugins: browser, memory-lancedb-pro, qqbot; ...)
```

## Unhealthy patterns

- `plugins.allow: plugin not installed: qqbot` → `openclaw plugins install @openclaw/qqbot`
- `plugin not installed: qqbot — install the official external plugin` → same fix
- No `[qqbot]` lines in journalctl → plugin not loaded, check install
- `WebSocket` connect fails → check appId/clientSecret, network to api.sgroup.qq.com
