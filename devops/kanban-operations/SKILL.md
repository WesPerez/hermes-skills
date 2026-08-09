---
name: kanban-operations
description: "Use for Hermes Kanban multi-agent operations: orchestrator decomposition, worker lifecycle, goal-mode cards, dependency links, recovery, and handoff discipline."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
environments: [kanban]
metadata:
  hermes:
    tags: [kanban, multi-agent, orchestration, workers, delegation, recovery]
    related_skills: [coding-agent-delegation]
---

# Kanban Operations

## Overview

Use this umbrella for Hermes Kanban work whether you are acting as an orchestrator creating cards or a worker executing a card. The key distinction is role: orchestrators decompose and route; workers execute their assigned card and produce verifiable handoffs.

## Role Split

| Role | Primary responsibility | Never do |
|---|---|---|
| Orchestrator | discover available profiles, split lanes, create dependency graph, route work | implement the work directly or invent profile names |
| Worker | claim one card, execute exactly its acceptance criteria, block or complete with evidence | spawn unrelated work or silently ignore blockers |

## Orchestrator Playbook

1. Discover available profiles (`hermes profile list` or ask the user).
2. Split the user request into independent lanes.
3. Link only real dependencies with parent relationships.
4. Create parent cards first, then children with `parents=[...]` so the dispatcher gates them.
5. For open-ended cards, use goal mode with explicit acceptance criteria.
6. Summarize the task graph with actual assignees and dependencies.

## Worker Playbook

1. Read the whole card, comments, parent summaries, and acceptance criteria.
2. Do the smallest complete unit of work that satisfies the card.
3. Use tools to verify, not just reason.
4. If blocked, call the kanban block path with a precise unblock request.
5. On completion, provide evidence: files changed, commands run, outputs/URLs/ids, residual risks.

## Dependency and Recovery Rules

- Parent first, child second. A child must not become ready before inputs exist.
- If a reviewer asks for changes, create a new implementation card linked from the review result rather than mutating history.
- Use reclaim/reassign/model-change recovery for stuck or crashing workers.
- Treat hallucinated card ids and unknown assignees as operational bugs to repair immediately.

## Common Pitfalls

- Bundling unrelated lanes into one card.
- Over-linking tasks because prose says "also" or "finally".
- Forgetting tenant/profile namespace when creating cards.
- Workers claiming success without an artifact or command output.
- Re-running the same failed worker without changing inputs, profile, or model.

## Verification Checklist

- [ ] Profiles exist before assignment
- [ ] Independent work can run in parallel
- [ ] Dependencies encoded structurally, not only in prose
- [ ] Workers return verifiable handles/output
- [ ] Recovery actions preserve audit trail
