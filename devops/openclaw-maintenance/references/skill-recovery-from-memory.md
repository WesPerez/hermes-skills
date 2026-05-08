# Recovering Deleted Skills from Memory Files

When a custom skill is deleted (e.g., during workspace cleanup before it was migrated), the skill's SKILL.md may still be recoverable from daily memory files.

## How to Find Lost Skills

### Step 1: Search memory files

```bash
cd ~/.openclaw/workspace/memory/
grep -l "新增技能\|skill\|SKILL.md\|edge-browser\|custom skill" *.md 2>/dev/null
```

Look for keywords like:
- `新增技能` / `新建技能` / `创建了`
- `SKILL.md` — file path references
- The skill name or purpose (e.g., `doubao`, `deepseek`, `画图`)

### Step 2: Extract the skill definition

Memory files typically document:
- Skill name and location
- Trigger words / invocation commands
- Step-by-step workflow
- Technical dependencies (browser, tools, ports)
- Known issues and fixes

Example from `2026-03-19.md`:

```
### 新增技能：edge-browser-agent-doubao ⭐⭐⭐
- 功能：控制服务器 Edge 浏览器访问豆包进行图像生成
- 触发词：豆包画图、豆包生成图片、doubao画图
- 技能位置：~/.openclaw/workspace/skills/edge-browser-agent-doubao/
- 图像生成流程：
  1. 打开豆包 → 点击"图像生成"按钮
  2. 输入详细提示词（建议200字以上）
  3. 按 Enter 发送，等待30-60秒生成
- 图片下载：豆包图片URL带有防盗链...
```

### Step 3: Reconstruct the SKILL.md

From the memory entry, reconstruct the skill:

```markdown
---
name: edge-browser-agent-doubao
description: 通过 Edge 浏览器自动访问豆包进行图像生成
read_when:
  - 用户要求画图/生图/生成图片
  - 触发词：豆包画图
---
# 豆包图像生成

## 依赖
- Edge 浏览器 + CDP 端口 9222（持久化 profile）
- Hermes browser 工具（CDP 模式）或 agent-browser CLI

## 工作流程
1. 打开豆包网站 → 点击"图像生成"按钮
2. 输入详细提示词（200字以上获得更好结果）
3. 按 Enter 发送，等待 30-60 秒生成

## 注意事项
- 默认使用 3:4 比例，裁剪后 288x361
- 提示词加入构图要求：`画面上方留出约30像素的空间，主体放在中下部`
- 自动裁剪顶部 23 像素水印
- 豆包图片 URL 有防盗链，直接 curl 获取高清版会返回 HTML 错误页
```

### When Recovery Fails

If the skill was never documented in memory files, try:

1. **Search OpenClaw workspace git history** — if the workspace was git-tracked before cleanup
2. **Debugfs filesystem recovery** — last resort for ext4 filesystems:
   ```bash
   debugfs -R "ls -d /path/to/deleted/directory" /dev/vda2
   ```
   (Only works if inodes haven't been reused)
3. **skills.sh / ClawHub** — check if a published alternative exists:
   ```bash
   npx skills find <keyword>
   ```
   Example: `npx skills find doubao` found `hunduncn/xpai-doubao-web@xpai-doubao-web` (201 installs)

## Prevention

Before deleting `workspace/skills/`, always:
1. List and review each skill
2. Check if it has a `.clawhub/origin.json` (published) or not (custom)
3. Offer to migrate custom skills to `~/.openclaw/skills/` or `~/.hermes/skills/`
4. Document the skill in a memory file as insurance
