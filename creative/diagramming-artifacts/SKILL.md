---
name: diagramming-artifacts
description: "Use for visual diagrams as artifacts: architecture/cloud/infra SVG/HTML diagrams and hand-drawn Excalidraw-style flow, sequence, and system diagrams."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [diagrams, architecture, svg, html, excalidraw, cloud, flowcharts]
    related_skills: [claude-design]
---

# Diagramming Artifacts

## Overview

Use this umbrella when the user asks for a diagram artifact rather than prose: architecture maps, cloud/infrastructure diagrams, flowcharts, sequence diagrams, or hand-drawn Excalidraw JSON.

## Format Choice

| Desired style | Output format |
|---|---|
| polished dark architecture/cloud/infra | self-contained HTML/SVG |
| hand-drawn collaborative diagram | Excalidraw JSON or exported image |
| quick flow/system sketch | HTML/SVG or Excalidraw depending on editability needs |

## Architecture / SVG Diagrams

- Use explicit nodes, groups, labels, arrows, legends, and data-flow direction.
- Prefer dark theme for infra diagrams unless the user specifies otherwise.
- Keep exported HTML self-contained and browser-openable.
- Start from `templates/architecture-diagram/template.html` when useful.

## Excalidraw Diagrams

- Use Excalidraw JSON for editable hand-drawn diagrams.
- Keep element ids stable enough for later edits.
- Use color and grouping intentionally; don't overload the hand-drawn style.
- Upload/export using `scripts/excalidraw/upload.py` when needed.

References migrated from `excalidraw` live under `references/excalidraw/`.

## Verification Checklist

- [ ] Diagram answers the intended question/flow
- [ ] Labels are readable at target size
- [ ] Arrows have clear direction and endpoints
- [ ] Artifact opens/imports without syntax errors
- [ ] File path and format are reported
