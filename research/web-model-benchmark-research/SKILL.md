---
name: web-model-benchmark-research
description: Research AI model coding benchmarks via browser — bypasses dynamic rendering truncation, extracts full benchmark tables from official blogs
triggers:
  - compare coding benchmarks
  - model benchmark numbers
  - SWE-bench LiveCodeBench terminal-bench
  - extract benchmark table from web page
---

# Web Model Benchmark Research Skill

## When to Use
Researching coding/programming benchmark data for AI models (LLMs) via web browsing. Replaces `search_files` for web content.

## Core Problem
Many model benchmark pages use dynamic rendering (React/Vue), causing `browser_snapshot` to truncate tables. `search_files` consistently returns 0 results in this environment for web content.

## Approach

### 1. Go to Official Sources First
- Model official blogs (e.g., `z.ai/blog/<model>`, `deepseek.com/blog`)
- HuggingFace model card pages (`huggingface.co/<org>/<model>`)
- Third-party综述文章 (DataCamp, articles summarizing releases)

**Avoid**: Raw GitHub README URLs — often return empty pages or uncrawlable content in this environment.

### 2. Extract Full Page Content with Offset Slicing
When `browser_snapshot` truncates at ~8000 chars, use `browser_console` to extract slices:

```javascript
// Extract middle section of page (bypass snapshot truncation)
document.body.innerText.slice(8000, 12000)

// Find specific section by keyword
document.body.innerText.slice(
  document.body.innerText.indexOf('KEYWORD_START'),
  document.body.innerText.indexOf('KEYWORD_END')
)
```

### 3. Get Benchmark Tables from Blogs
Model benchmark tables are often embedded as:
- Static text in the blog body (search by benchmark name)
- SVG charts with embedded text values
- Full-width tables below the main narrative

Use `browser_console` to extract:
```javascript
// Get all text matching a pattern
document.body.innerText.match(/SWE-Bench[A-Za-z0-9 .—\-/%()]+/g)

// Find position of table in page
document.body.innerText.indexOf('Benchmark')
```

### 4. Check Multiple Pages for Same Model
Different sources may report different benchmarks:
- Official blog: detailed narrative + custom benchmarks
- HuggingFace model card: standardized comparison table (but often truncated)
- Third-party articles: cross-model comparison tables (often most complete)

## Known Model Benchmark Pages (Verified Working)
- GLM-5.1: `https://z.ai/blog/glm-5.1` — full benchmark table embedded in blog, ~12K+ chars total
- DeepSeek V4: `https://www.datacamp.com/blog/deepseek-v4` — structured comparison table

## Terminal-Bench 2.0 Harnesses
This benchmark has multiple evaluation frameworks. When comparing models:
- **Terminus-2**: standard harness
- **Claude Code**: model uses external AI coding agent
- **Codex**: another harness variant

Results are NOT directly comparable across harnesses. Always note which harness was used.

## Known Issue: DeepSeek V3.2 in GLM-5.1 Benchmarks
In GLM-5.1's comparison table, DeepSeek-V3.2 appears but shows "—" for many coding benchmarks (SWE-Bench Pro, NL2Repo). Do NOT assume DeepSeek V3.2 was not tested — the model may simply not have participated in that evaluation round. Cross-reference with DeepSeek's own benchmark pages.
