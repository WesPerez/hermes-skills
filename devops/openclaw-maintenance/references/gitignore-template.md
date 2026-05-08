# OpenClaw .gitignore — track config + skills, exclude runtime + secrets
# Usage: cp to ~/.openclaw/, then git add .gitignore openclaw.json skills/ workspace/

# ── 密钥和凭证 ──
.env
agents/

# ── 运行时数据 ──
canvas/
cron/
exec-approvals.json
extensions/
identity/
logs/
memory/
npm/
plugins/
plugin-skills/
qqbot/
tasks/
update-check.json

# ── 自动备份和临时文件 ──
*.bak
*.last-good
*.tmp

# ── workspace 运行时 ──
workspace/plugins/
workspace/.clawhub/
workspace/.learnings/
workspace/.openclaw/
workspace/.claude/
workspace/HEARTBEAT.md

# ── 技能运行时/敏感数据 ──
skills/*/.clawhub/
skills/*/.clawdhub/
skills/*/.learnings/
skills/*/_meta.json
skills/colg-hotlist/cookies.json
