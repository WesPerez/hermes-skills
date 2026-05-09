---
name: openclaw-maintenance
description: OpenClaw config cleanup, workspace maintenance, plugin verification, and post-cleanup validation.
read_when:
  - Cleaning up or simplifying OpenClaw config
  - Removing unused providers, channels, or skills from OpenClaw
  - OpenClaw QQ Bot (or other channel) stops responding after config changes
  - Verifying plugin install state after upgrades or cleanup
---

# OpenClaw Maintenance & Cleanup

Config cleanup, workspace maintenance, and plugin verification for OpenClaw (clawbot).

## Pre-cleanup: Snapshot current state

```bash
openclaw plugins list
ls ~/.openclaw/npm/node_modules/@openclaw/  # installed npm plugins
ls -la ~/.openclaw/skills/                   # managed skills
ls -la ~/.openclaw/workspace/skills/          # workspace skills (if any)
```

## Config cleanup: Remove unused providers/channels

Edit `~/.openclaw/openclaw.json` with `python3 -c "import json; ..."` — OpenClaw validates schemas strictly.

**Keep only what's needed.** For a QQ-only bot with DeepSeek:
- `models.providers`: only `deepseek` with `deepseek-v4-pro`
- `agents.defaults.models`: only `deepseek/deepseek-v4-pro`
- `bindings`: only `qqbot`
- `channels`: only `qqbot`
- `auth.profiles`: only `deepseek:default`
- `plugins.allow`: `["memory-lancedb-pro", "qqbot", "browser", "deepseek"]`

Remove everything else: `volcengine`, `minimax`, `doubao`, fallback configs, unused channels.

## ⚠️ Workspace cleanup — check for custom skills FIRST

**🚨 CRITICAL: Before deleting `workspace/skills/`, check if it contains custom user-created skills.**

The directory may contain custom skills that were hand-authored for specific workflows (e.g. browser automation for specific websites, custom integrations). These are **not** recoverable from clawhub/skills.sh — they were never published.

```bash
# Check for custom skills
ls -la ~/.openclaw/workspace/skills/ 2>/dev/null

# Check if any skill has a custom origin (not from clawhub or skills.sh)
for d in ~/.openclaw/workspace/skills/*/; do
  [ -f "$d/.clawhub/origin.json" ] && echo "❌ PUBLISHED: $d" || echo "✅ CUSTOM: $d"
done
```

**If custom skills exist:**
1. **Offer to migrate them** to `~/.openclaw/skills/` or `~/.hermes/skills/` instead of deleting
2. If user confirms deletion, **check if the skill content was documented in `workspace/memory/YYYY-MM-DD.md`** — many custom skills have detailed step-by-step records in daily memory files that can be used to reconstruct the skill
3. Only after confirming there's nothing to save should you delete `workspace/skills/`

**Custom skills I've seen deleted this way (real casualties):**
- `edge-browser-agent-doubao` — browser automation for 豆包 image generation (recoverable from `workspace/memory/2026-03-19.md`)
- `edge-browser-agent-deepseek` — same pattern for DeepSeek querying

Delete these from `~/.openclaw/workspace/`:
- `skills/` — old agent skills not needed for QQ bot (⚠️ ONLY after checking for custom skills)
- `.agents/` — orphan symlink target dir (skills that symlinked here are already dead)
- `.claude/skills/` — Claude-specific skills, same orphan issue
- `memory/evolution/` — old GEP evolution cycles
- `memory/cron-states/`, `memory/cron-*` — stale cron state files

**Keep:**
- `AGENTS.md`, `SOUL.md`, `USER.md`, `MEMORY.md`, `SHORTCUTS.md` — agent identity
- `memory/YYYY-MM-DD.md` — daily notes
- `memory/ontology/graph.jsonl` — if referenced by SOUL.md shortcuts
- `plugins/memory-lancedb-pro/` — if loaded via `plugins.load.paths` in config

Also clean `~/.openclaw/skills/` — managed skills may have broken symlinks to deleted `.agents/skills/`:

```bash
# Find and remove broken symlinks
find ~/.openclaw/skills/ -type l ! -exec test -e {} \; -print -delete
```

## 🚨 QQ Bot "no response" — diagnostic cheat sheet

When QQ Bot suddenly stops responding to messages, check in THIS ORDER:

```bash
# 1. IS THE PLUGIN INSTALLED? (most common cause — 80% of cases)
openclaw plugins list 2>&1 | grep qqbot
# If output is empty or says "plugin not installed" → openclaw plugins install @openclaw/qqbot

# 2. IS THE SERVICE RUNNING?
systemctl --user is-active openclaw-gateway
# If dead → systemctl --user restart openclaw-gateway

# 3. IS THE WEBSOCKET CONNECTED?
journalctl --user -u openclaw-gateway --no-pager --since "5 minutes ago" | grep -iE "qqbot.*websocket|qqbot.*ready|qqbot.*token"
# Expected: "WebSocket connected", "Access token obtained", "Gateway ready"

# 4. IS THE MODEL CONFIG VALID?
journalctl --user -u openclaw-gateway --no-pager --since "5 minutes ago" | grep -iE "unknown model|failover"
# If model errors → check models.providers in openclaw.json
```

**Most common root cause:** `~/.openclaw/npm/` directory was cleaned, removing `@openclaw/qqbot`. The npm plugin is installed to `~/.openclaw/npm/node_modules/@openclaw/qqbot/`. After `openclaw plugins install @openclaw/qqbot`, restart the gateway.

## CRITICAL: Verify plugins after cleanup

**After any cleanup that touches `~/.openclaw/npm/`, verify QQ Bot is installed:**

```bash
openclaw plugins list | grep qqbot
```

If missing:
```bash
openclaw plugins install @openclaw/qqbot
```

This will create a new `openclaw.json.bak` — delete it:
```bash
rm ~/.openclaw/openclaw.json.bak
```

## Restart and verify

```bash
systemctl --user restart openclaw-gateway
sleep 4
journalctl --user -u openclaw-gateway --no-pager --since "30 seconds ago" | grep -iE "qqbot|error|ready|websocket"
```

**Expected healthy output:**
```
✅ Access token obtained successfully
WebSocket connected
Gateway ready / Gateway resumed
3 plugins: browser, memory-lancedb-pro, qqbot
```

**Check for warnings:**
```bash
openclaw plugins list 2>&1 | grep -i "warning\\|not installed\\|not found" || echo "✅ No plugin warnings"
```

## Official source repo (for diffing against upstream)

OpenClaw's running code is the npm-compiled JS at `/usr/lib/node_modules/openclaw/` (no `.git`).
To compare config against the official defaults or track upstream changes, clone the source repo:

```bash
git clone --depth 1 https://github.com/openclaw/openclaw.git ~/openclaw-repo
```

Unlike Hermes (`pip install -e .` → the git clone IS the running code), OpenClaw separates:
- **Source**: `~/openclaw-repo` (TypeScript, official GitHub)
- **Runtime**: `/usr/lib/node_modules/openclaw/` (compiled JS, npm-installed)
- **Config**: `~/.openclaw/` (JSON config, local git repo)

## Hermes model_normalize.py: when to revert local patches

The official Hermes v0.12.0 already handles `deepseek-v4-pro` in its normalization white-list.
If another agent (Claude Code, etc.) applied a local patch to `hermes_cli/model_normalize.py`
that changes the default fallback from `deepseek-chat` to `deepseek-v4-pro`, it is redundant
and should be reverted to keep the tree clean:

```bash
cd ~/hermes-agent-repo
git checkout -- hermes_cli/model_normalize.py
```

Verify the official version still passes:
```bash
python3 -c "
from hermes_cli.model_normalize import normalize_model_for_provider
assert normalize_model_for_provider('deepseek-v4-pro', 'deepseek') == 'deepseek-v4-pro'
assert normalize_model_for_provider('deepseek-v4-flash', 'deepseek') == 'deepseek-v4-flash'
print('✅ OK')
"
```

Only keep genuinely useful local patches (e.g. `weixin.py` auto-QR-relogin).
Drop the old stash after verifying: `git stash drop`.

## Git tracking for config history

Set up `~/.openclaw` as a git repo so config changes are diffable:

```bash
cd ~/.openclaw
git init
```

Create `.gitignore` — see `references/gitignore-template.md` for the full template.
Key exclusions: `.env`, `agents/` (API keys), `qqbot/`, `cron/`, `memory/`,
`logs/`, `npm/`, `plugins/`, `tasks/`, `*.bak`, `*.last-good`, `*.tmp`,
`workspace/plugins/`, `skills/*/cookies.json`, `skills/*/.clawhub/`.

```bash
git add .gitignore openclaw.json skills/ workspace/
git commit -m "initial: OpenClaw config snapshot"
```

After any config change:
```bash
cd ~/.openclaw && git diff  # review what changed
git add -A && git commit -m "描述改动"
```

## Workspace cleanup — extended exclusions

In `~/.openclaw/workspace/memory/`, also remove:
- `cron-wrapper.sh`, `cron-status.json`, `cron-states/`
- `evolution/` — old GEP cycles
- `tasks-log.md`, `knowledge-base.jsonl`, `evolver_update_check.json`

In `~/.openclaw/skills/`, exclude from git tracking (add to `.gitignore`):
- `skills/*/.clawhub/` — runtime skill hub metadata
- `skills/*/.clawdhub/` — same, alternate spelling
- `skills/*/.learnings/` — runtime learning logs
- `skills/*/_meta.json` — auto-generated metadata
- `skills/colg-hotlist/cookies.json` — session cookies (contains auth data)

## ⚠️ Deleting Claude Code traces — STOP BEFORE ROOT

When cleaning up another AI agent's traces (Claude Code run as `claudeuser` or root):

**🔴 ABSOLUTE RULE: Never delete root's own config files without explicit user confirmation.**

This includes:
- `~/.claude.json` — contains API keys the user configured, session history, cost data. **NEVER `rm -rf` this.**
- `~/.claude/` — session history, settings, backups. May contain important config the user needs.
- Any `.env` or credential file belonging to root — even if it looks like "agent data"

**Safe to delete without asking:**
- `/home/claudeuser/` — dedicated agent account, entirely disposable
- `/tmp/claude-*`, `/tmp/openclaw-1003/` — temp files
- `.nvm/.../lib/node_modules/@anthropic-ai/claude-code/` — npm package (reinstallable)
- `/run/sudo/ts/1003` — sudo timestamp for deleted user

**Requires user confirmation first:**
- `~/.claude.json` — ask before deleting
- `~/.claude/` — ask before deleting
- `.nvm/.../bin/claude` — the binary symlink (user may still need it)
- `/root/.local/share/applications/claude-*` — desktop entries

**After reinstalling Claude Code binary:**
```bash
npm install -g @anthropic-ai/claude-code
hash -r          # bash caches command paths — clear it
claude --version # verify it finds the new path
```
Without `hash -r`, bash may keep showing `No such file or directory` for the old nvm path even after reinstall. The `hash -r` step is NOT optional — it's the fix for "I reinstalled but it still can't find it".

## WorkBuddy vs OpenClaw

WorkBuddy and OpenClaw are related but different — see `references/workbuddy-vs-openclaw.md` for the full comparison. Key takeaway: **WorkBuddy is the desktop GUI (Win/Mac only), OpenClaw is the CLI tool (Linux-friendly).** Don't confuse the two when setting up or troubleshooting.

## Common pitfalls

- **`openclaw plugins install` creates `openclaw.json.bak`** — delete it after install, you don't need it.
- **Broken symlinks in `~/.openclaw/skills/` cause "Skipping escaped skill path" log noise but don't break functionality. Clean them: `find ~/.openclaw/skills/ -type l ! -exec test -e {} \\; -delete`.**
- **Escaping symlinks** (point outside the configured root) generate the same log noise but are NOT caught by the broken-symlink check. Find and remove them:
  ```bash
  for f in ~/.openclaw/skills/*; do
    [ -L "$f" ] && realpath "$f" | grep -q "^$HOME/.openclaw/skills/" || echo "REMOVE: $f"
  done
  ```
  The log pattern is: `Skipping escaped skill path outside its configured root: source=openclaw-managed reason=symlink-escape requested=... resolved=...`
- **`memory-lancedb-pro` plugin loaded from workspace** — if `plugins.load.paths` points to `~/.openclaw/workspace/plugins/memory-lancedb-pro`, do NOT delete that directory during workspace cleanup.
- **Cleaning `~/.openclaw/npm/` kills QQ Bot** — `@openclaw/qqbot` is an external npm plugin installed to `~/.openclaw/npm/node_modules/@openclaw/qqbot/`. Never delete `npm/` directory. If gone, reinstall: `openclaw plugins install @openclaw/qqbot`.
- **`workspace/plugins/memory-lancedb-pro` may have nested `.git`** — delete it if broken: `rm -rf ~/.openclaw/workspace/plugins/memory-lancedb-pro/.git`.
- **Config validation is strict** — always use `python3 -c "json.load(open(...))"` to validate before restart. A bad config will prevent gateway startup.
- **`systemctl --user` not `systemctl`** — OpenClaw runs as a user service.
- **Investigating Claude Code changes** — if another AI agent was active on the machine, see `references/claude-code-investigation.md` for a step-by-step forensic guide (user home, .claude.json, session-env, bash_history, modified files).
