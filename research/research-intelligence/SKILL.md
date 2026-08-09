---
name: research-intelligence
description: "Use for research intelligence workflows: arXiv discovery, blog/RSS monitoring, LLM wiki knowledge bases, Polymarket data, and benchmark/domain reconnaissance."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [research, arxiv, rss, blogs, knowledge-base, polymarket, benchmarks]
    related_skills: [research-paper-writing, web-access]
---

# Research Intelligence

## Overview

Use this umbrella for gathering and maintaining research intelligence: finding papers, monitoring feeds, querying prediction markets, building a markdown knowledge base, or collecting benchmark/domain evidence. It covers discovery and analysis inputs; use `research-paper-writing` when producing a submission manuscript.

## Routing Table

| Need | Section |
|---|---|
| Find academic papers | arXiv discovery |
| Monitor blogs/RSS | Blogwatcher |
| Build/query interlinked markdown KB | LLM Wiki |
| Prediction market data | Polymarket |
| Benchmark/domain reconnaissance | Web/model benchmark research notes |

## arXiv Discovery

- Search by keyword, author, category, or id.
- Capture title, authors, abstract, date, url, and why it is relevant.
- Use scripts migrated under `scripts/arxiv/` where useful.

## Blogwatcher

- Use for recurring or one-shot feed monitoring.
- Prefer RSS/Atom over scraping when available.
- Summaries should separate new facts from interpretation.

## LLM Wiki / Knowledge Base

- Use for building linked markdown notes from source material.
- Keep citation/source links in each note.
- Prefer concise topic pages with backlinks over large undifferentiated dumps.

## Polymarket

- Use for markets, prices, orderbooks, and history.
- Report market ids/slugs, timestamps, and whether values are prices, probabilities, or liquidity.
- Reference material migrated under `references/polymarket/`.

## Verification Checklist

- [ ] Source type and query are explicit
- [ ] URLs/ids/timestamps captured
- [ ] Claims are separated from extracted source facts
- [ ] Outputs are saved or summarized in a reusable form
