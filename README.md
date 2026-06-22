# 🦞 Hermes 技能仓库

用户自定义 Hermes Agent 技能集合，共 **115** 个技能。
每个技能是一个 SKILL.md 文件，按分类存放，用 Git 做版本控制。

## 搜索前缀触发

| 前缀 | 功能 | 技能 |
|------|------|------|
| `ds xxx` | DeepSeek 专家模式搜索 | ds-expert |
| `db xxx` | 豆包 AI 图像生成 | doubao-image-gen |
| `colg xxx` | COLG 论坛热榜 | colg-hotlist |
| 其他 | 普通联网搜索 | web-access（三层通道） |

---

## 🐛 调试

- **debug-js-syntax-errors** — 调试单文件 HTML 应用中 JavaScript 语法错误，特别是中文嵌套引号问题。
- **gateway-message-delivery-debug** — 诊断 Hermes Gateway 消息截断、静默投递失败和各平台 API 限制（微信、Telegram、Discord 等）。
- **hermes-reasoning-deepseek** — 诊断并修复 Hermes Agent 中 DeepSeek 思考/推理配置不生效的问题。
- **openclaw-model-fix** — 修复 OpenClaw 中 Unknown model 错误，通过添加缺失的模型 ID 到 providers 配置中。

## ⚙️ 运维

- **kanban-orchestrator** — 看板工作流编排器：任务分解、专家分配、进度跟踪，负责将大任务拆解为子任务分派给看板工人。
- **kanban-worker** — 看板工人执行规范：任务生命周期、注意事项和边界案例，被编排器调用的执行端。
- **openclaw-maintenance** — OpenClaw 配置清理、工作区维护、插件验证和清理后验证工具。
- **remote-desktop-setup** — 在无头服务器上搭建远程桌面（Xfce4 + VNC + NoVNC），配合 nginx 反向代理和 Basic Auth 用于浏览器自动化。
- **webhook-subscriptions** — Webhook 订阅管理：事件驱动的自动化 Agent 运行。

## 🎮 娱乐

- **find-nearby** — 使用 OpenStreetMap 查找附近地点（餐厅、咖啡厅、药店等），无需 API Key。

## 🔍 搜索

- **doubao-image-gen** — 用户消息以 db 开头时，通过 Edge CDP 控制浏览器进入豆包网页版进行 AI 图像生成并返回图片。
- **ds-expert** — 用户消息以 ds/DS 开头时，通过 Edge CDP 进入 DeepSeek 专家模式搜索，提取答案并保留原始格式返回。
- **web-access** — 三层通道联网调度：搜索发现→内容抓取→CDP浏览器，处理所有搜索、网页抓取、登录后操作等联网任务。

## 🍎 Apple

- **apple-notes** — 通过 memo CLI 管理 Apple Notes：创建、搜索、编辑。
- **apple-reminders** — 通过 remindctl 管理 Apple 提醒事项：添加、列出、完成。
- **findmy** — 通过 macOS FindMy.app 追踪 Apple 设备/AirTags。
- **imessage** — 通过 imsg CLI 在 macOS 上发送和接收 iMessages/SMS。

## 🤖 自主Agent

- **claude-code** — 将编码任务委托给 Claude Code CLI 执行（功能开发、PR 创建）。
- **codex** — 将编码任务委托给 OpenAI Codex CLI 执行（功能开发、PR 创建）。
- **hermes-agent** — 配置、扩展或贡献 Hermes Agent 本身。
- **opencode** — 将编码任务委托给 OpenCode CLI 执行（功能开发、代码审查）。

## 🎨 创意

- **architecture-diagram** — 以 HTML 格式生成深色主题 SVG 架构/云/基础设施图。
- **ascii-art** — ASCII 艺术生成：pyfiglet、cowsay、boxes、图片转 ASCII。
- **ascii-video** — ASCII 视频：将视频/音频转换为彩色 ASCII MP4/GIF。
- **baoyu-comic** — 知识漫画创作：教育、传记、教程类漫画。
- **baoyu-infographic** — 信息图生成：21 种布局 x 21 种风格的信息可视化图表。
- **claude-design** — 设计一次性 HTML 作品（落地页、演示文稿、原型）。
- **comfyui** — 使用 ComfyUI 生成图像、视频和音频：安装、启动、管理节点模型、运行工作流。
- **creative-ideation** — Generate project ideas via creative constraints.
- **design-md** — 编写/验证/导出 Google DESIGN.md 令牌规范文件。
- **excalidraw** — 手绘风格 Excalidraw JSON 图表（架构图、流程图、时序图）。
- **humanizer** — 人性化文本处理：去除 AI 痕迹，添加真实语气。
- **manim-video** — Manim CE 动画：3Blue1Brown 风格的数学/算法视频。
- **p5js** — p5.js 创作：生成艺术、着色器、交互式、3D 作品。
- **pixel-art** — 像素艺术制作，支持多种时代调色板（NES、Game Boy、PICO-8）。
- **popular-web-designs** — 54 个真实设计系统（Stripe、Linear、Vercel 等）的 HTML/CSS 复刻。
- **pretext** — 构建 DOM 无关的文本布局演示：ASCII 艺术、文本几何游戏、动态排版。
- **sketch** — 快速 HTML 原型：一次性 2-3 个设计变体对比。
- **songwriting-and-ai-music** — 歌曲创作技巧和 Suno AI 音乐提示词生成。
- **touchdesigner-mcp** — 通过 MCP 控制 TouchDesigner：创建算子、设置参数、连接节点、运行实时视觉。

## 📈 数据科学

- **jupyter-live-kernel** — 通过实时 Jupyter 内核进行迭代式 Python 数据探索。

## 📧 邮件

- **himalaya** — 终端邮件客户端：IMAP/SMTP 收发邮件。

## 🎮 游戏

- **minecraft-modpack-server** — 搭建 Mod 版 Minecraft 服务器（支持 CurseForge、Modrinth）。
- **pokemon-player** — 通过无头模拟器 + 内存读取玩宝可梦游戏。

## 🐙 GitHub

- **codebase-inspection** — 使用 pygount 检查代码库：代码行数、语言占比、文件比例。
- **github-auth** — GitHub 认证配置：HTTPS Token、SSH 密钥、gh CLI 登录。
- **github-code-review** — PR 代码审查：查看差异、内联评论，通过 gh 或 REST API 提交。
- **github-issues** — 通过 gh 或 REST API 创建、分类、标记、分配 GitHub Issue。
- **github-pr-workflow** — GitHub PR 生命周期管理：分支、提交、PR 创建、CI 检查、合并。
- **github-repo-management** — 克隆/创建/Fork 仓库；管理远程地址和发布版本。

## 🔌 MCP

- **mcporter** — 使用 mcporter CLI 列出、配置、认证和调用 MCP 服务器/工具。
- **native-mcp** — MCP 客户端：连接服务器、注册工具（stdio/HTTP 模式）。

## 🎵 媒体

- **gif-search** — 通过 Tenor API 搜索/下载 GIF（curl + jq）。
- **heartmula** — HeartMuLa：类似 Suno 的歌词+标签生成歌曲。
- **songsee** — 音频频谱图/特征分析（梅尔谱、色度图、MFCC）。
- **spotify** — Spotify 控制：播放、搜索、排队、管理播放列表和设备。
- **youtube-content** — YouTube 字幕转摘要、讨论帖、博客文章。

## 🤖 MLOps

- **audiocraft** — AudioCraft: MusicGen text-to-music, AudioGen text-to-sound.
- **axolotl** — Axolotl：YAML 配置的 LLM 微调（LoRA、DPO、GRPO）。
- **clip** — OpenAI CLIP 模型：零样本图像分类、图文匹配、跨模态检索。
- **dspy** — DSPy：声明式 LM 程序，自动优化提示，RAG 流水线。
- **gguf** — GGUF format and llama.cpp quantization for efficient CPU/GPU inference. Use when deploying models...
- **grpo-rl-training** — GRPO/RL 强化学习微调训练指导，使用 TRL 框架。
- **guidance** — 使用 Microsoft Guidance 框架控制 LLM 输出结构（JSON、XML、代码）。
- **huggingface-hub** — HuggingFace hf CLI：搜索/下载/上传模型和数据集。
- **llama-cpp** — llama.cpp 本地 GGUF 推理 + HuggingFace Hub 模型发现。
- **lm-evaluation-harness** — lm-eval-harness: benchmark LLMs (MMLU, GSM8K, etc.).
- **modal** — Serverless GPU cloud platform for running ML workloads. Use when you need on-demand GPU access wi...
- **obliteratus** — OBLITERATUS：通过 diff-in-means 方法消除 LLM 拒绝行为。
- **outlines** — Outlines：结构化 JSON/正则/Pydantic LLM 输出生成。
- **peft** — Parameter-efficient fine-tuning for LLMs using LoRA, QLoRA, and 25+ methods. Use when fine-tuning...
- **pytorch-fsdp** — PyTorch FSDP 全分片数据并行训练：参数分片、混合精度、CPU 卸载。
- **segment-anything** — SAM: zero-shot image segmentation via points, boxes, masks.
- **stable-diffusion** — State-of-the-art text-to-image generation with Stable Diffusion models via HuggingFace Diffusers....
- **trl-fine-tuning** — TRL: SFT, DPO, PPO, GRPO, reward modeling for LLM RLHF.
- **unsloth** — Unsloth：2-5x 更快的 LoRA/QLoRA 微调，更低显存占用。
- **vllm** — vLLM: high-throughput LLM serving, OpenAI API, quantization.
- **weights-and-biases** — W&B：ML 实验日志、超参搜索、模型注册、仪表板。
- **whisper** — OpenAI Whisper 语音识别：99 种语言、转录、翻译。

## 📝 笔记

- **obsidian** — 读取、搜索、创建和编辑 Obsidian 知识库笔记。

## 📊 效率

- **airtable** — Airtable REST API：记录增删改查、过滤、更新导入。
- **google-workspace** — Gmail、日历、云盘、文档、表格管理（gws CLI 或 Python）。
- **linear** — Linear：通过 GraphQL 和 curl 管理 Issue、项目、团队。
- **maps** — 通过 OpenStreetMap/OSRM 进行地理编码、POI 搜索、路线规划。
- **nano-pdf** — 编辑 PDF 文本/标题，通过 nano-pdf CLI 使用自然语言指令。
- **notion** — Notion API：页面、数据库、块、搜索管理。
- **ocr-and-documents** — 从 PDF/扫描件中提取文本（pymupdf、marker-pdf）。
- **powerpoint** — 创建、读取、编辑 .pptx 演示文稿、幻灯片、备注、模板。

## 🔴 红队

- **godmode** — LLM 越狱技术：Parseltongue、GODMODE、ULTRAPLINIAN 等。

## 📚 研究

- **arxiv** — 通过关键词、作者、分类或 ID 搜索 arXiv 论文。
- **blogwatcher** — 通过 blogwatcher-cli 监控博客和 RSS/Atom 订阅源。
- **chinese-llm-research** — 调研中国大模型（DeepSeek、GLM、Qwen 等）的基准数据和技术规格。
- **llm-wiki** — Karpathy 的 LLM Wiki：构建/查询互联的 Markdown 知识库。
- **polymarket** — 查询 Polymarket：市场、价格、订单簿、历史数据。
- **research-paper-writing** — 撰写 ML 论文（NeurIPS/ICML/ICLR），从设计到投稿全流程。
- **web-model-benchmark-research** — 通过浏览器研究 AI 模型编码基准评测，绕过动态渲染截断，提取完整基准表格。

## 🏠 智能家居

- **openhue** — 通过 OpenHue CLI 控制 Philips Hue 灯、场景、房间。

## 📱 社交

- **xitter** — 通过 x-cli 终端客户端与 X/Twitter 交互：发帖、搜索、私信等。
- **xurl** — 通过 xurl CLI 操作 X/Twitter：发帖、搜索、私信、媒体、v2 API。

## 💻 开发

- **debugging-hermes-tui-commands** — 调试 Hermes TUI 斜杠命令：Python、Gateway、Ink UI。
- **hermes-agent-skill-authoring** — 编写 SKILL.md 技能文件：YAML 前置元数据、验证器、目录结构规范。
- **node-inspect-debugger** — 通过 --inspect + Chrome DevTools Protocol CLI 调试 Node.js。
- **plan** — 计划模式：编写 Markdown 计划到 .hermes/plans/，不执行代码。
- **python-debugpy** — Python 调试：pdb REPL + debugpy 远程调试（DAP）。
- **requesting-code-review** — 提交前代码审查：安全扫描、质量门禁、自动修复。
- **spike** — 一次性实验：在构建前验证某个想法的可行性。
- **subagent-driven-development** — 通过 delegate_task 子 Agent 执行计划（两阶段审查）。
- **systematic-debugging** — 四阶段根因调试：先理解 Bug 再修复。
- **test-driven-development** — TDD：严格遵循 RED-GREEN-REFACTOR，测试先于代码。
- **writing-plans** — 编写实现计划：分解为小任务、路径和代码。

## 📂 其他

- **colg-hotlist** — 用户消息以 colg 开头时，查看 COLG 论坛社区热榜，提取热帖列表并截图发送。
- **colg-reply** — COLG 论坛自动回复帖子，使用 Hermes 浏览器工具自动化回复操作。
- **dogfood** — Web 应用的探索性 QA 测试：发现 Bug、收集证据、生成报告。
- **frontend-debug-checks** — 调试 HTML 内联脚本中的 JS 语法错误、CSS 特指度问题和常见静默失败。
- **yuanbao** — 元宝群组操作：@提及用户、查询信息/成员。

---
📅 2026-05-08 自动生成 | 🧾 117 个技能 | 🌐 全中文描述
