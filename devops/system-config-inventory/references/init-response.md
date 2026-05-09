# ⚙️ 系统配置总览

## 🤖 Hermes Agent (v0.12.0)
- 提供者: glm-5.1 / custom
- Dashboard: http://43.159.168.34/hermes/ (admin/admin)
- 微信网关: 运行中

## 🔌 OpenClaw (clawbot v2026.5.6)
### QQ Bot
- appId: `102870423`
- clientSecret: `2eHuYCrXDubJ1kTDyjVH4rfTI7xofXPI`
- imageServerBaseUrl: `http://127.0.0.1:18789`
- chatId: `E81071DFBD3E34593174A0E70827CE88`
- 插件: `@openclaw-china/qqbot`
- 安装: `openclaw plugins install @openclaw-china/qqbot`
- GitHub: https://github.com/BytePioneer-AI/openclaw-china

### 企业微信
- corpId: `ww7d2f702fcd93439a`
- corpSecret: `a1i_9OELvR7dfOsT1vizQpviiXLAChyy-2YtlnnVjN8`
- agentId: `1000002`
- encodingAesKey: `8XaVMOC6JF9PFyYP4YPglCeUd7RX07Eo6daHTP6fwIs`
- 插件: `@openclaw-china/wecom`
- 安装: `openclaw plugins install @openclaw-china/wecom`

### Gateway
- 端口: `18789`
- 模式: local / bind=lan
- auth_token: `5d4ae7...056c`

### Plugin列表
| 插件 | 用途 |
|------|------|
| memory-lancedb-pro | 长期记忆 (workspace) |
| qqbot | QQ通道 |
| browser | 浏览器自动化 |
| deepseek | DeepSeek推理 |

## 🌐 网络服务
- nginx: 80(HTTP) / 8888(HTTPS)
- NoVNC桌面: http://43.159.168.34/desktop/ (admin/admin, VNC密码hermes123)
- Edge CDP: ws://127.0.0.1:9222 (持久化profile)
- 化学游戏: http://43.159.168.34/chemistry-game.html

## 📁 系统文件路径
- Hermes配置: `~/.hermes/config.yaml`
- Hermes技能: `~/.hermes/skills/` (git)
- OpenClaw配置: `~/.openclaw/openclaw.json`
- OpenClaw环境变量: `~/.openclaw/.env`
- OpenClaw记忆: `~/.openclaw/workspace/memory/`
- OpenClaw知识图谱: `~/.openclaw/workspace/memory/ontology/graph.jsonl`

## 📋 OpenClaw已装技能
capability-evolver, proactive-agent, find-skills, github, gog, ontology, self-improving-agent, skill-vetter, summarize, tavily-search, weather

## 🔄 Workspace
- QQ agent: `main`
- 共享workspace: `~/.openclaw/workspace`
- 记忆/文件/规则共享，认证/模型配置独立
