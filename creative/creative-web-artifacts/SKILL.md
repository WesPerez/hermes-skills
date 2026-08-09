---
name: creative-web-artifacts
description: "Use for browser-based creative artifacts: HTML mockups/prototypes, brand-inspired web designs, DESIGN.md token specs, p5.js sketches, Pretext text demos, and interactive visual experiments."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [creative, html, design, prototype, p5js, pretext, design-system]
    related_skills: [diagramming-artifacts]
---

# Creative Web Artifacts

## Overview

Use this umbrella for designed or generative browser artifacts: one-off HTML pages, prototypes, sketch variants, brand-inspired design systems, DESIGN.md token specs, p5.js generative art, and Pretext text-as-geometry demos.

## Routing Table

| Need | Section |
|---|---|
| Designed one-off page/prototype/deck | Design process |
| Throwaway variants to compare | Sketch variants |
| Known brand visual vocabulary | Popular web designs |
| Formal design token spec | DESIGN.md |
| Generative art / shaders / interactive sketches | p5.js |
| Text layout as geometry / ASCII-like web demos | Pretext |

## Design Process

- Gather context before inventing visuals: brand, screenshots, repo tokens, target audience.
- Avoid generic AI design tropes and filler content.
- Produce local files and verify they open cleanly.
- Original `claude-design` doctrine is preserved under `references/absorbed-skills/claude-design.md`.

## Sketch Variants

- Use for fast comparison boards: conservative, strong-fit, divergent.
- Keep variants comparable; do not make them mere color swaps.

## Popular Web Designs

- Use as visual vocabulary, not cloning. Transform principles into original artifacts.
- Templates migrated under `templates/popular-web-designs/`.

## DESIGN.md

- Use when the output is a persistent token/design-system spec rather than a rendered artifact.
- Starter template migrated under `templates/design-md/`.

## p5.js and Pretext

- p5.js: canvas/generative art, shaders, interaction, WebGL, exports.
- Pretext: DOM-free text layout, typographic flow, text-as-geometry demos.
- Detailed references/templates/scripts migrated under prefixed directories.

## Verification Checklist

- [ ] Artifact format chosen intentionally
- [ ] Source context inspected when available
- [ ] File exists and syntax/browser console checked when possible
- [ ] Design/generative choices explained briefly
