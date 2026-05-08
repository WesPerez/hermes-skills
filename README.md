# Hermes Skills

用户自定义 Hermes Agent 技能仓库。
每个技能对应一个 SKILL.md 文件，按分类存放。
当前共 **116** 个技能。

---

## Debugging

- **debug-js-syntax-errors** — Debug JavaScript syntax errors in single-file HTML apps with inline <script> tags, espe...
- **gateway-message-delivery-debug** — Systematic approach for diagnosing message truncation, silent delivery failures, and pl...
- **hermes-reasoning-deepseek** — Diagnose and fix DeepSeek thinking/reasoning configuration in Hermes Agent — when reaso...
- **html-game-timer-debug** — Debug double-counting bugs in single-file HTML games where timer callbacks fire after a...
- **openclaw-model-fix** — Fix Unknown model errors in OpenClaw by adding missing model IDs to models.providers co...

## Devops

- **kanban-orchestrator** — Decomposition playbook + specialist-roster conventions + anti-temptation rules for an o...
- **kanban-worker** — Pitfalls, examples, and edge cases for Hermes Kanban workers. The lifecycle itself is a...
- **openclaw-maintenance** — OpenClaw config cleanup, workspace maintenance, plugin verification, and post-cleanup v...
- **remote-desktop-setup** — Set up a remote desktop (Xfce4 + VNC + NoVNC) on a headless server with nginx reverse p...
- **webhook-subscriptions** — Webhook subscriptions: event-driven agent runs.

## Leisure

- **chemistry-game-debug** — 化学游戏 chemistry-game.html 调试与修复手册
- **find-nearby** — Find nearby places (restaurants, cafes, bars, pharmacies, etc.) using OpenStreetMap. Wo...

## Search

- **doubao-image-gen** — 用户消息以"db"开头时，通过 Edge CDP 控制浏览器进入豆包网页版进行 AI 图像生成并返回图片。
- **ds-expert** — 用户消息以"ds/DS"开头时，通过 Edge CDP 进入 DeepSeek 专家模式搜索，提取答案并保留原始格式返回。

## Apple

- **apple-notes** — Manage Apple Notes via memo CLI: create, search, edit.
- **apple-reminders** — Apple Reminders via remindctl: add, list, complete.
- **findmy** — Track Apple devices/AirTags via FindMy.app on macOS.
- **imessage** — Send and receive iMessages/SMS via the imsg CLI on macOS.

## Autonomous Ai Agents

- **claude-code** — Delegate coding to Claude Code CLI (features, PRs).
- **codex** — Delegate coding to OpenAI Codex CLI (features, PRs).
- **hermes-agent** — Configure, extend, or contribute to Hermes Agent.
- **opencode** — Delegate coding to OpenCode CLI (features, PR review).

## Creative

- **architecture-diagram** — Dark-themed SVG architecture/cloud/infra diagrams as HTML.
- **ascii-art** — ASCII art: pyfiglet, cowsay, boxes, image-to-ascii.
- **ascii-video** — ASCII video: convert video/audio to colored ASCII MP4/GIF.
- **baoyu-comic** — Knowledge comics (知识漫画): educational, biography, tutorial.
- **baoyu-infographic** — Infographics: 21 layouts x 21 styles (信息图, 可视化).
- **claude-design** — Design one-off HTML artifacts (landing, deck, prototype).
- **comfyui** — Generate images, video, and audio with ComfyUI — install, launch, manage nodes/models, ...
- **creative-ideation** — Generate project ideas via creative constraints.
- **design-md** — Author/validate/export Google's DESIGN.md token spec files.
- **excalidraw** — Hand-drawn Excalidraw JSON diagrams (arch, flow, seq).
- **humanizer** — Humanize text: strip AI-isms and add real voice.
- **manim-video** — Manim CE animations: 3Blue1Brown math/algo videos.
- **p5js** — p5.js sketches: gen art, shaders, interactive, 3D.
- **pixel-art** — Pixel art w/ era palettes (NES, Game Boy, PICO-8).
- **popular-web-designs** — 54 real design systems (Stripe, Linear, Vercel) as HTML/CSS.
- **pretext** — Use when building creative browser demos with @chenglou/pretext — DOM-free text layout ...
- **sketch** — Throwaway HTML mockups: 2-3 design variants to compare.
- **songwriting-and-ai-music** — Songwriting craft and Suno AI music prompts.
- **touchdesigner-mcp** — Control a running TouchDesigner instance via twozero MCP — create operators, set parame...

## Data Science

- **jupyter-live-kernel** — Iterative Python via live Jupyter kernel (hamelnb).

## Email

- **himalaya** — Himalaya CLI: IMAP/SMTP email from terminal.

## Gaming

- **minecraft-modpack-server** — Host modded Minecraft servers (CurseForge, Modrinth).
- **pokemon-player** — Play Pokemon via headless emulator + RAM reads.

## Github

- **codebase-inspection** — Inspect codebases w/ pygount: LOC, languages, ratios.
- **github-auth** — GitHub auth setup: HTTPS tokens, SSH keys, gh CLI login.
- **github-code-review** — Review PRs: diffs, inline comments via gh or REST.
- **github-issues** — Create, triage, label, assign GitHub issues via gh or REST.
- **github-pr-workflow** — GitHub PR lifecycle: branch, commit, open, CI, merge.
- **github-repo-management** — Clone/create/fork repos; manage remotes, releases.

## Mcp

- **mcporter** — Use the mcporter CLI to list, configure, auth, and call MCP servers/tools directly (HTT...
- **native-mcp** — MCP client: connect servers, register tools (stdio/HTTP).

## Media

- **gif-search** — Search/download GIFs from Tenor via curl + jq.
- **heartmula** — HeartMuLa: Suno-like song generation from lyrics + tags.
- **songsee** — Audio spectrograms/features (mel, chroma, MFCC) via CLI.
- **spotify** — Spotify: play, search, queue, manage playlists and devices.
- **youtube-content** — YouTube transcripts to summaries, threads, blogs.

## Mlops

- **audiocraft** — AudioCraft: MusicGen text-to-music, AudioGen text-to-sound.
- **axolotl** — Axolotl: YAML LLM fine-tuning (LoRA, DPO, GRPO).
- **clip** — OpenAI's model connecting vision and language. Enables zero-shot image classification, ...
- **dspy** — DSPy: declarative LM programs, auto-optimize prompts, RAG.
- **gguf** — GGUF format and llama.cpp quantization for efficient CPU/GPU inference. Use when deploy...
- **grpo-rl-training** — Expert guidance for GRPO/RL fine-tuning with TRL for reasoning and task-specific model ...
- **guidance** — Control LLM output with regex and grammars, guarantee valid JSON/XML/code generation, e...
- **huggingface-hub** — HuggingFace hf CLI: search/download/upload models, datasets.
- **llama-cpp** — llama.cpp local GGUF inference + HF Hub model discovery.
- **lm-evaluation-harness** — lm-eval-harness: benchmark LLMs (MMLU, GSM8K, etc.).
- **modal** — Serverless GPU cloud platform for running ML workloads. Use when you need on-demand GPU...
- **obliteratus** — OBLITERATUS: abliterate LLM refusals (diff-in-means).
- **outlines** — Outlines: structured JSON/regex/Pydantic LLM generation.
- **peft** — Parameter-efficient fine-tuning for LLMs using LoRA, QLoRA, and 25+ methods. Use when f...
- **pytorch-fsdp** — Expert guidance for Fully Sharded Data Parallel training with PyTorch FSDP - parameter ...
- **segment-anything** — SAM: zero-shot image segmentation via points, boxes, masks.
- **stable-diffusion** — State-of-the-art text-to-image generation with Stable Diffusion models via HuggingFace ...
- **trl-fine-tuning** — TRL: SFT, DPO, PPO, GRPO, reward modeling for LLM RLHF.
- **unsloth** — Unsloth: 2-5x faster LoRA/QLoRA fine-tuning, less VRAM.
- **vllm** — vLLM: high-throughput LLM serving, OpenAI API, quantization.
- **weights-and-biases** — W&B: log ML experiments, sweeps, model registry, dashboards.
- **whisper** — OpenAI's general-purpose speech recognition model. Supports 99 languages, transcription...

## Note Taking

- **obsidian** — Read, search, create, and edit notes in the Obsidian vault.

## Productivity

- **airtable** — Airtable REST API via curl. Records CRUD, filters, upserts.
- **google-workspace** — Gmail, Calendar, Drive, Docs, Sheets via gws CLI or Python.
- **linear** — Linear: manage issues, projects, teams via GraphQL + curl.
- **maps** — Geocode, POIs, routes, timezones via OpenStreetMap/OSRM.
- **nano-pdf** — Edit PDF text/typos/titles via nano-pdf CLI (NL prompts).
- **notion** — Notion API via curl: pages, databases, blocks, search.
- **ocr-and-documents** — Extract text from PDFs/scans (pymupdf, marker-pdf).
- **powerpoint** — Create, read, edit .pptx decks, slides, notes, templates.

## Red Teaming

- **godmode** — Jailbreak LLMs: Parseltongue, GODMODE, ULTRAPLINIAN.

## Research

- **arxiv** — Search arXiv papers by keyword, author, category, or ID.
- **blogwatcher** — Monitor blogs and RSS/Atom feeds via blogwatcher-cli tool.
- **chinese-llm-research** — 调研中国大模型（DeepSeek、智谱GLM、通义Qwen等）的基准数据、模型信息、技术规格的标准方法论
- **llm-wiki** — Karpathy's LLM Wiki: build/query interlinked markdown KB.
- **polymarket** — Query Polymarket: markets, prices, orderbooks, history.
- **research-paper-writing** — Write ML papers for NeurIPS/ICML/ICLR: design→submit.
- **web-model-benchmark-research** — Research AI model coding benchmarks via browser — bypasses dynamic rendering truncation...

## Smart Home

- **openhue** — Control Philips Hue lights, scenes, rooms via OpenHue CLI.

## Social Media

- **xitter** — Interact with X/Twitter via the x-cli terminal client using official X API credentials....
- **xurl** — X/Twitter via xurl CLI: post, search, DM, media, v2 API.

## Software Development

- **debugging-hermes-tui-commands** — Debug Hermes TUI slash commands: Python, gateway, Ink UI.
- **hermes-agent-skill-authoring** — Author in-repo SKILL.md: frontmatter, validator, structure.
- **node-inspect-debugger** — Debug Node.js via --inspect + Chrome DevTools Protocol CLI.
- **plan** — Plan mode: write markdown plan to .hermes/plans/, no exec.
- **python-debugpy** — Debug Python: pdb REPL + debugpy remote (DAP).
- **requesting-code-review** — Pre-commit review: security scan, quality gates, auto-fix.
- **spike** — Throwaway experiments to validate an idea before build.
- **subagent-driven-development** — Execute plans via delegate_task subagents (2-stage review).
- **systematic-debugging** — 4-phase root cause debugging: understand bugs before fixing.
- **test-driven-development** — TDD: enforce RED-GREEN-REFACTOR, tests before code.
- **writing-plans** — Write implementation plans: bite-sized tasks, paths, code.

## Uncategorized

- **colg-hotlist** — 用户消息以"colg"开头时，查看 COLG 论坛社区热榜，提取热帖列表并截图发送。
- **colg-reply** — COLG论坛自动回复帖子。使用 Hermes 浏览器工具自动化回复。
- **dogfood** — Exploratory QA of web apps: find bugs, evidence, reports.
- **frontend-debug-checks** — Debug JS syntax errors in inline HTML script blocks, CSS specificity issues, and common...
- **yuanbao** — Yuanbao (元宝) groups: @mention users, query info/members.

---
自动生成 | 2026-05-08
