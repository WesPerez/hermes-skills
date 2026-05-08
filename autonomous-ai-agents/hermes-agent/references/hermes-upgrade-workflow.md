# Hermes Agent Upgrade Workflow

Complete procedure for upgrading Hermes Agent between versions, with pitfalls.

## Pre-Upgrade Checklist

1. Note current version: `hermes --version`
2. Note current model config: `grep -A3 "^model:" ~/.hermes/config.yaml`
3. Stash local source changes: `cd ~/hermes-agent-repo && git stash push -m "pre-upgrade-$(date +%Y%m%d)"`
4. Check what changed locally: `git stash show --stat`
5. Verify gateway is healthy: `systemctl --user status hermes-gateway`

## Upgrade Steps

```bash
# 1. Pull latest code
cd ~/hermes-agent-repo
git fetch origin
git reset --hard origin/main

# 2. Check if local patches survived
git stash show --stat
# Compare stashed files vs new version:
git diff HEAD..stash@{0} -- <file>
# If official code has equivalent fixes, drop stash. Otherwise re-apply.

# 3. Update Python dependencies
~/.hermes-venv/bin/python -m ensurepip --upgrade  # if pip missing
~/.hermes-venv/bin/python -m pip install -e . --upgrade

# 4. Migrate config
hermes config migrate

# 5. Restart gateway
systemctl --user restart hermes-gateway
```

## Post-Upgrade Verification

```bash
hermes --version
systemctl --user status hermes-gateway --no-pager | head -5

# Critical for DeepSeek: verify model normalization
python3 -c "
from hermes_cli.model_normalize import normalize_model_for_provider
r = normalize_model_for_provider('deepseek-v4-pro', 'deepseek')
assert r == 'deepseek-v4-pro', f'FAIL: got {r}'
print(f'OK: {r}')
"

hermes config check
hermes doctor | head -20

# Check WeChat connectivity
grep "weixin.*connect\|weixin.*error" ~/.hermes/logs/agent.log | tail -5

# Verify actual model used
grep "Auxiliary auto-detect" ~/.hermes/logs/agent.log | tail -3
```

## Common Pitfalls

### Pip not installed in venv
`~/.hermes-venv/bin/python -m ensurepip --upgrade`

### DeepSeek silently remapped to flash (pre-0.12.0)
`deepseek-v4-pro` was normalized to `deepseek-chat`, which DeepSeek backend maps to `deepseek-v4-flash`. Always run the normalization check.

### Stash conflicts on platform adapters
Local modifications to `gateway/platforms/` may conflict with new versions. Check the official version's support before re-applying.

### Config version mismatch
Run `hermes config migrate` to add new default settings (curator, providers, etc.).

### Drain timeout on restart
Normal when an active session was in progress. New instance starts clean.
