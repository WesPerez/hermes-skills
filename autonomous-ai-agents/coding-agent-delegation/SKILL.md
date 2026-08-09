---
name: coding-agent-delegation
description: "Use to delegate coding work to external autonomous coding CLIs (Claude Code, Codex, OpenCode) while Hermes owns scoping, verification, review, and final reporting."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [autonomous-agents, coding, claude-code, codex, opencode, delegation, review]
    related_skills: [subagent-driven-development, github-workflows]
---

# Coding Agent Delegation

## Overview

Use this umbrella when an external coding CLI should implement, review, or explore code while Hermes remains the accountable orchestrator. Claude Code, Codex, and OpenCode are interchangeable delegation lanes with different strengths and local availability constraints.

## Tool Selection

| CLI | Best fit | Notes |
|---|---|---|
| Claude Code | broad feature work, repo-aware edits, PR-style tasks | often strong at large-context code navigation |
| Codex CLI | focused implementation, patch generation, experiments | good isolated lane; pair with Hermes verification |
| OpenCode | implementation and code review using OpenCode ecosystem | useful as independent second opinion |

Prefer the CLI the user has configured. If multiple are available, choose the one that best matches the task and budget; for high-risk changes, run a second CLI as a reviewer rather than letting one agent self-certify.

## Hermes Ownership Rules

1. Hermes scopes the task and passes exact paths, constraints, tests, and acceptance criteria.
2. The delegated agent may edit files, but Hermes must inspect `git diff` and run verification afterwards.
3. Do not report success from the delegate's self-report alone.
4. Keep delegated prompts bounded: one goal, explicit non-goals, expected output, and test command.
5. Preserve user constraints and language/tone in the delegated prompt.

## Typical Workflow

```bash
# Example shape; adapt to the installed CLI.
claude-code "Implement ... Acceptance: ... Run: ..."
codex "Fix ... in paths ... Do not ..."
opencode "Review the diff for correctness/security/tests ..."
```

After the CLI exits:

```bash
git status --short
git diff --stat
git diff
# run the project-specific tests/build/lint
```

## Review Lane Pattern

For significant changes:

1. Implementation lane: one CLI makes the change.
2. Independent review lane: another CLI or Hermes reviews the diff.
3. Hermes reconciles feedback, applies fixes, and reruns tests.
4. Final response names real verification output.

## Common Pitfalls

- Do not leave long-running interactive CLIs unattended without PTY/background handling.
- Do not delegate vague tasks like "fix everything"; external CLIs over-edit.
- Do not accept generated tests that merely encode the current broken behavior.
- Do not let a delegated agent push, merge, or publish without explicit user intent.
- Do not expose secrets in prompts copied into third-party CLIs.

## Verification Checklist

- [ ] CLI availability/auth checked
- [ ] Prompt contains exact acceptance criteria and test command
- [ ] `git diff` inspected after delegate exits
- [ ] Tests/build/lint actually run by Hermes
- [ ] Final answer distinguishes delegate claims from verified facts
