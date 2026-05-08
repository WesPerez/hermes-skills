# OpenClaw Upgrade & Cleanup Workflow

## Pre-Upgrade

```bash
openclaw version
npm view openclaw version
systemctl --user status openclaw-gateway
```

## Upgrade Steps

```bash
# Use official registry if tencent mirror fails
npm install -g openclaw@latest --registry https://registry.npmjs.org

# Install required plugins (QQ Bot is now a separate npm plugin)
openclaw plugins install @openclaw/qqbot

# plugins install creates a .bak — delete it
rm -f ~/.openclaw/openclaw.json.bak

# Remove old TS-source extensions that conflict
rm -rf ~/.openclaw/extensions/qqbot

# Restart
systemctl --user restart openclaw-gateway
```

## Post-Upgrade Verification

```bash
# Version
openclaw version

# Gateway status
systemctl --user status openclaw-gateway --no-pager | head -5

# Model config
journalctl --user -u openclaw-gateway --no-pager --since "30s" | grep "agent model:"

# QQ Bot connectivity
journalctl --user -u openclaw-gateway --no-pager --since "30s" | grep -i qqbot | grep -E "connected|resumed|ready|error"

# Verify all plugins loaded (qqbot MUST be present)
openclaw plugins list 2>&1 | grep -E "qqbot.*enabled" || echo "❌ QQ Bot plugin NOT loaded!"

# No model errors
journalctl --user -u openclaw-gateway --no-pager --since "1m" | grep -iE "unknown model|failover" || echo "OK"
```

## Cleanup

Remove accumulated backup/config junk after upgrade:

```bash
# openclaw.json backups (can accumulate dozens)
rm -f ~/.openclaw/openclaw.json.bak*

# Clobbered configs from version conflicts
rm -f ~/.openclaw/openclaw.json.clobbered*

# Old tar backups
rm -f /root/openclaw-backup-*.tar.gz

# Text dumps
rm -f ~/.openclaw/memory-lancedb-pro.txt
rm -f ~/.openclaw/year-report.txt

# Hermes config backups
rm -f ~/.hermes/config.yaml.bak*

# Verify clean
find ~/.openclaw -maxdepth 1 \( -name "*.bak*" -o -name "*.clobbered*" \) | wc -l
```

## Common Pitfalls

### npm registry issues with tencent mirror
The tencent mirror (`mirrors.tencentyun.com/npm`) may fail for specific dependencies (e.g., `pac-resolver`). Use `--registry https://registry.npmjs.org`.

### QQ Bot plugin requires reinstall in v2026.5.x+
The QQ Bot adapter moved from built-in TypeScript extension to separate npm plugin. Old extension dir at `~/.openclaw/extensions/qqbot` must be deleted.

### Config file watch triggers backup
OpenClaw auto-creates backup files on config changes. After upgrade, these accumulate. Safe to delete after verification.
