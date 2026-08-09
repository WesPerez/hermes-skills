---
name: github-workflows
description: "Use for GitHub operations as one lifecycle: authentication, repository setup, issues, PR creation/CI/merge, and PR/code review via gh, git, curl, or REST/GraphQL."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [github, git, gh, pull-requests, issues, code-review, repositories, ci]
    related_skills: [codebase-inspection]
---

# GitHub Workflows

## Overview

Use this umbrella when a task touches GitHub: setting up authentication, cloning or creating repositories, managing issues, opening and updating PRs, reviewing diffs, monitoring CI, or merging. Treat GitHub work as one connected lifecycle rather than separate micro-skills.

## Decision Map

| Need | Use this section |
|---|---|
| `gh`/token/SSH/HTTPS auth fails | Authentication |
| Clone/fork/create repo, remotes, releases | Repository management |
| Create/triage/label/close tickets | Issues |
| Branch, commit, push, PR, CI, merge | Pull request lifecycle |
| Review PRs, inline comments, REST fallback | Code review |

## Authentication

1. Prefer `gh auth status` when `gh` is installed.
2. If `gh` is unavailable, discover `GITHUB_TOKEN` from environment, `${HERMES_HOME:-$HOME/.hermes}/.env`, or `~/.git-credentials`.
3. For API calls use:
   ```bash
   curl -H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" ...
   ```
4. For git remotes, support both `https://github.com/owner/repo.git` and `git@github.com:owner/repo.git`.

Support material migrated from `github-auth` lives under `scripts/github-auth/`.

## Repository Management

- Extract `OWNER/REPO` from `git remote get-url origin` before REST calls.
- Use `gh repo clone`, `gh repo create`, `gh repo fork`, `gh release ...` where available.
- Fall back to REST API for repo metadata, releases, branch protection, and collaborators.
- Keep remote mutations explicit and report the exact repo URL after creation or fork.

Reference: `references/github-repo-management/github-api-cheatsheet.md`.

## Issues

Use issues for durable task tracking, bug reports, feature requests, and triage.

Typical flow:

```bash
gh issue create --title "..." --body-file issue.md --label bug
# fallback
curl -X POST -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/issues \
  -d '{"title":"...","body":"...","labels":["bug"]}'
```

Templates migrated from `github-issues` live under `templates/github-issues/`.

## Pull Request Lifecycle

1. Start from a clean updated base branch.
2. Create a named branch (`feat/...`, `fix/...`, `docs/...`, `refactor/...`).
3. Commit with a useful conventional message.
4. Push and create a PR with a summary and test plan.
5. Monitor CI (`gh pr checks --watch`, `gh run view --log-failed`, or REST fallbacks).
6. Fix failures in a loop, pushing incremental commits.
7. Merge only when checks and review requirements are satisfied.

References/templates migrated from `github-pr-workflow` live under `references/github-pr-workflow/` and `templates/github-pr-workflow/`.

## Code Review

Review PRs systematically:

- Inspect diff and changed files, not just the PR description.
- Check security, correctness, tests, migration/backward compatibility, and operational risks.
- Prefer actionable comments with file/line context.
- Use `gh pr diff`, `gh pr view --json ...`, `gh pr review`, or REST review endpoints.
- Return a concise verdict: approve/comment/request changes, with blockers first.

Reference: `references/github-code-review/review-output-template.md`.

## Package Notes

This umbrella absorbed former standalone GitHub skills. Their support files were re-homed under prefixed directories so links are stable and discoverable from this skill root.

## Verification Checklist

- [ ] Auth method selected (`gh` or token fallback)
- [ ] `OWNER/REPO` resolved from remote or URL
- [ ] Mutating actions confirmed when appropriate
- [ ] PR/issue/repo URL or API status reported
- [ ] CI/review state verified before claiming merge readiness
