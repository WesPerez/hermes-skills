---
name: development-quality-loop
description: "Use for software development quality workflows: spike experiments, root-cause debugging, language debuggers, TDD, pre-commit review, and simplification/cleanup."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [software-development, debugging, tdd, review, refactor, quality]
    related_skills: [writing-plans, subagent-driven-development]
---

# Development Quality Loop

## Overview

Use this umbrella when improving code quality through a disciplined loop: investigate, reproduce, test, fix, review, and simplify. It replaces narrow workflow skills for spikes, systematic debugging, language-specific debugger attachment, TDD, pre-commit verification, and parallel cleanup.

## Workflow Map

| Situation | Section |
|---|---|
| Unknown feasibility or API behavior | Spike experiments |
| Bug exists but root cause unclear | Systematic debugging |
| Need interactive Python or Node inspection | Language debugger recipes |
| Feature/bugfix should be test-first | Test-driven development |
| Before committing or publishing | Pre-commit review |
| Recent changes feel overcomplicated | Simplification pass |

## Spike Experiments

- Make experiments throwaway and explicitly scoped.
- Write down the hypothesis, fastest probe, success/failure signal, and cleanup plan.
- Do not let spike code silently become production code without review.

## Systematic Debugging

1. Reproduce the failure and capture exact symptoms.
2. Localize the component and recent changes.
3. Form hypotheses and test them one at a time.
4. Patch the smallest root cause fix.
5. Add or update regression tests.

## Language Debugger Recipes

- Python: use `pdb` for local step-through or `debugpy` for DAP/remote attach.
- Node: use `node --inspect`/`--inspect-brk` and Chrome DevTools Protocol when needed.
- Always verify debugger ports, process ids, and that breakpoints actually bind.

Original detailed recipes are preserved under `references/absorbed-skills/`.

## Test-Driven Development

- RED: write or expose a failing test for the intended behavior.
- GREEN: make the smallest change that passes.
- REFACTOR: improve structure while preserving passing tests.
- Avoid tests that merely lock in broken behavior.

## Pre-Commit Review

Before finalizing code:

- Inspect `git diff` and changed files.
- Run relevant test/build/lint commands.
- Check security, error handling, backwards compatibility, migrations, and docs.
- Fix issues and rerun verification.

## Simplification Pass

Use after a feature lands or a messy bugfix stabilizes:

- Remove dead paths and duplicated logic.
- Collapse unnecessary abstractions.
- Prefer readable boring code over clever scaffolding.
- Keep behavior fixed by tests while refactoring.

## Verification Checklist

- [ ] Failure or objective is explicit
- [ ] Tests or probes prove the behavior
- [ ] Debugger/spike artifacts are cleaned up or documented
- [ ] Diff reviewed for security and maintainability
- [ ] Final response includes real command output or limitations
