# Hermes 技能仓库

这是 `~/.hermes/skills` 的 Git 追溯仓库，用于保存当前可用的 Hermes Agent 技能。技能数量和名称以磁盘上的 `SKILL.md` 为准，不在 README 中维护容易失真的静态全量清单。

## 目录约定

- `<category>/<skill>/SKILL.md`：分类技能。
- `<skill>/SKILL.md`：少量根级技能。
- `references/`、`scripts/`、`templates/`：技能需要的配套资料与工具。
- `references/absorbed-skills/`：curator 合并后保留的旧技能正文，便于追溯。

## 技能来源

仓库中的内容分为四类，提交时不要混淆来源：

1. **Hermes bundled**：来自 `/root/hermes-agent-repo/skills`。同步时应先做逐字节比对，再作为独立提交记录。
2. **Hub packages**：由 Hermes Hub 安装。提交前使用 `.hub/lock.json` 的 `content_hash` 校验目录完整性；`.hub/` 本身是运行台账，不入库。
3. **Local skills**：为本机工作流编写的技能，应有明确触发条件、操作步骤、风险边界和验证方式。
4. **Curator umbrellas**：curator 将多个窄技能合并后的 umbrella。旧内容应迁入 `references/`，不要只删除原技能而丢失知识。

## 核心路由

| 输入前缀或场景 | 技能 |
|---|---|
| `ds ...` | `search/ds-expert` |
| `db ...` | `search/doubao-image-gen` |
| `colg ...` | `colg-hotlist` |
| 其他联网搜索与网页访问 | `search/web-access` |

当前主要 umbrella 包括 `apple-ecosystem`、`coding-agent-delegation`、`kanban-operations`、`github-workflows`、`creative-web-artifacts`、`ai-audio-production`、`llm-model-operations`、`note-taking-tools`、`research-intelligence` 和 `development-quality-loop`。

## 提交边界

以下内容属于运行状态或本地恢复数据，已由 `.gitignore` 排除，不得提交：

- `.usage.json`、`.usage.json.lock`
- `.curator_state`、`.curator_backups/`
- `.archive/`
- `.hub/`
- `.bundled_manifest`

同步或整理时按来源拆分提交。不要把 bundled 更新、Hub 包、本地技能和 curator 迁移塞进同一个提交。

## 快速审查

列出当前活跃技能：

```bash
find . -path './.*' -prune -o -name SKILL.md -type f -print | sort
```

统计活跃技能：

```bash
find . -path './.*' -prune -o -name SKILL.md -type f -print | wc -l
```

检查重复的 frontmatter 名称：

```bash
find . -path './.*' -prune -o -name SKILL.md -type f -exec awk '/^name:/{print $2; exit}' {} \; | sort | uniq -d
```

提交前至少检查：

```bash
git status --short
git diff --check
```

## 恢复

已提交内容优先从 Git 历史恢复。curator 的临时归档位于本机 `.archive/`，但它不是长期备份，也不应替代 Git 追溯。
