# WorkBuddy vs OpenClaw（龙虾）

两个产品都是腾讯系 AI Agent 生态的一部分，但形态不同，容易混淆。

## 对比

| 维度 | WorkBuddy | OpenClaw (clawbot) |
|------|-----------|-------------------|
| **本质** | 桌面 GUI 客户端 | CLI 命令行工具 |
| **官方名称** | 腾讯 WorkBuddy（腾讯版龙虾） | OpenClaw（开源龙虾） |
| **平台支持** | **macOS** (.dmg) + **Windows** (.exe) | **Linux**、macOS、Windows |
| **是否开源** | 否（闭源桌面应用） | 是（GitHub: openclaw/openclaw） |
| **安装方式** | 官网下载安装包双击安装 | `npm install -g openclaw` |
| **交互方式** | 图形界面，菜单驱动 | 终端命令 + systemd 服务 |
| **典型用途** | 办公自动化、报告生成、数据整理 | 服务器端机器人、自动化流程 |
| **Skill 系统** | 兼容 OpenClaw Skills | 原生 Skills 系统 |
| **渠道接入** | 微信/企微/飞书/钉钉等 IM（通过服务端） | 同左（通过 gateway） |

## Linux 支持情况

**WorkBuddy 官方没有 Linux 原生版本。** 官网仅提供：
- macOS: .dmg（Apple Silicon + Intel）
- Windows: .exe（x64，兼容 ARM64）

如果需要在 Linux 上使用 WorkBuddy 的能力，有两种变通方案：

1. **通过 WSL（Windows Subsystem for Linux）** — 在 Windows 上安装 WSL，在 WSL 内运行 OpenClaw CLI，通过 WorkBuddy 桌面端来管理
2. **直接使用 OpenClaw（clawbot）** — 在 Linux 上安装 `npm install -g openclaw`，完全 CLI 操作，本服务器就是这种方式

## 常见误解

- ❌ "WorkBuddy 就是 OpenClaw 的桌面版" — **不完全准确**。WorkBuddy 是闭源商业产品，OpenClaw 是开源 CLI。WorkBuddy 兼容 OpenClaw 的 Skills 生态，但底层实现不同。
- ❌ "服务器上装 WorkBuddy" — 服务器上应该装 OpenClaw，不是 WorkBuddy。
- ✅ 两者的 Skills、配置、模型互通。同一个 `SKILL.md` 可以在 WorkBuddy 和 OpenClaw 上运行。

## 腾讯文档 Open API 频率限制

通过 WorkBuddy 连接腾讯文档时的 API 限制（来源：DeepSeek 专家模式查询）：

| 维度 | 限制 | 说明 |
|------|------|------|
| **按文档 (fileID)** | 150 次/分钟 | 同一份文档的操作 |
| **按用户 (openID)** | 300 次/分钟 | 跨所有文档的总量 |
| **导入导出** | 9 次/天 | 按用户维度，严格限制 |

## 参考链接

- WorkBuddy 官网：https://www.codebuddy.cn/work/
- OpenClaw GitHub：https://github.com/openclaw/openclaw
