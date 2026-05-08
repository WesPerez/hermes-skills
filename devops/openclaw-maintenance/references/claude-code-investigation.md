# Investigating Claude Code / AI Agent Changes

When another AI agent (Claude Code, Cursor, Copilot, etc.) has been running on the
same machine and you need to find out what it changed:

## 1. Check if a dedicated user was created

```bash
awk -F: '$3>=1000{print $1, $3, $6}' /etc/passwd  # find new users
stat /home/<user> | grep Birth                      # when was it created
```

## 2. Find Claude Code traces

```bash
# Claude Code home directory
ls -la /home/<user>/.claude/
ls -la /home/<user>/.claude.json     # config + session stats + cost

# Conversation history
cat /home/<user>/.claude/history.jsonl   # each line = one user message + sessionId

# Session environments (model used, project path)
ls /home/<user>/.claude/session-env/

# Shell commands they ran
cat /home/<user>/.bash_history

# Session logs (binary/jsonl, may be empty if sessions expired)
ls -la /home/<user>/.claude/sessions/
```

## 3. Understand Claude Code's .claude.json

Key fields:
- `projects.<path>.lastSessionFirstPrompt` — what the user asked
- `projects.<path>.lastModelUsage` — which models were used and their token/cost breakdown
- `projects.<path>.lastCost` — total USD cost of the session
- `projects.<path>.lastDuration` — session duration in ms

## 4. Check what Claude Code's OWN OpenClaw looks like

Claude Code often sets up its own OpenClaw instance under the user:
```bash
cat /home/<user>/.openclaw/openclaw.json | python3 -m json.tool | head -30
# Note: often uses a completely different model/provider than root's OpenClaw
```

## 5. Find files Claude Code modified on root filesystem

```bash
# Files modified after the user was created
find /root/.openclaw /root/.hermes -newer /home/<user> -type f 2>/dev/null

# Or by timestamp range
find / -newer /home/<user> -not -path '/proc/*' -not -path '/sys/*' -type f 2>/dev/null
```

## 6. Check for Claude Code's own git changes on Hermes

```bash
cd ~/hermes-agent-repo
git status --short          # any modified files?
git diff --stat             # what files?
git stash list              # did it create a stash?
```

## Common Claude Code behaviors

- Runs under a dedicated user (e.g. `claudeuser`)
- Uses `--dangerously-skip-permissions` flag to access root filesystem
- Its `.openclaw/` config may differ completely from root's (different model, provider, auth)
- Creates temporary project directories under `.claude/projects/`
- Can leave `.bak` files in `~/.openclaw/` from `openclaw plugins install`
- May also leave traces under root: `/root/.claude.json`, `/root/.claude/` if Claude Code was run as root first

## 7. Aggressive cleanup: remove all Claude Code traces

When the user wants EVERYTHING deleted ("删干净", "全删了"):

**⚠️ WARNING: Step 4 removes `~/.claude.json` which contains the user's Anthropic API key.**
**ONLY delete root's Claude Code config if the user EXPLICITLY confirms they're done with it.**
**If in doubt, skip step 4 and 6 — keep the binary + config, just clean the dedicated agent user.**

```bash
# 1. Kill processes
pkill -u claudeuser

# 2. Delete user account + home directory
userdel -r claudeuser

# 3. Clean /tmp traces
rm -rf /tmp/openclaw-1003 /tmp/claude-1003 /tmp/claude-0 /tmp/openclaw-gw.log

# 4. ⚠️ ONLY WITH EXPLICIT USER CONFIRMATION — deletes API key config!
# rm -rf /root/.claude.json /root/.claude/

# 5. Clean sudo timestamp residue
rm -f /run/sudo/ts/1003

# 6. ⚠️ ONLY if user confirms they don't need Claude Code:
# npm uninstall -g @anthropic-ai/claude-code
# rm -rf /root/.nvm/versions/node/v*/lib/node_modules/@anthropic-ai/claude-code
# rm -f /root/.nvm/versions/node/v*/bin/claude

# 7. Remove desktop handler (safe, auto-regenerated)
rm -f /root/.local/share/applications/claude-code-url-handler.desktop

# 8. Final verification — should return 0
find / -path /proc -prune -o -path /sys -prune -o \( -user 1003 -o -group 1003 \) -print 2>/dev/null | wc -l
```

### If you accidentally deleted Claude Code binary/config:

```bash
# Reinstall binary
npm install -g @anthropic-ai/claude-code

# Clear bash command cache (old nvm path may be cached)
hash -r

# Verify
claude --version

# API key must be reconfigured — it was in the deleted .claude.json
# Option A: environment variable
echo 'export ANTHROPIC_API_KEY="sk-ant-..."' >> ~/.bashrc
source ~/.bashrc

# Option B: config file
mkdir -p ~/.claude
echo '{"primaryApiKey": "sk-ant-..."}' > ~/.claude.json
```
