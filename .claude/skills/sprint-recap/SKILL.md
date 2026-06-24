---
name: sprint-recap
description: Generate a sprint recap from a Notion sprint board and git history, write it to a Notion page, and create a GitHub PR with release notes
when_to_use: Use when the user asks to generate or update a sprint recap
argument-hint: "[notion-sprint-board-url] [notion-recap-page-url]"
allowed-tools: Bash(git log *) Bash(git diff *) Bash(git branch *) Bash(gh pr *) Bash(wc *) Grep Read
---

# Sprint Recap Generator

Generate a comprehensive sprint recap by cross-referencing a Notion sprint board with git history, then:
1. Write the recap to a Notion page
2. Create a GitHub PR from the staging branch to the production branch with the recap as release notes

> **Project-specific context** (sprint board URL, recap page ID, branch names, staging URL, task ID prefix) should be defined in the project's `CLAUDE.md` under `## Skill Configuration`. Check there first before asking the user.

## Inputs

Check the project's `CLAUDE.md` for defaults, then ask the user for anything still missing:
- **Sprint board URL** — Notion database with the sprint tasks
- **Column/filter** — which column or status group to pull tasks from (e.g. "Current Sprint", "In Progress")
- **Recap page URL** — Notion page where the recap should be written
- **Staging URL** — if the project has a staging environment, include it in the recap
- **Branch names** — staging branch and production branch (defaults: `staging` → `main`)

## Steps

### 1. Gather sprint tasks from Notion

Fetch the sprint board database. For each task in the target column, fetch the full card to get:
- Task ID (e.g. PROJ-123)
- Title
- Status (Done / In progress / Ready / Not ready)
- Assigned to
- Any implementation notes already on the card

### 2. Gather git history

For each task, search git for related branches and commits:

```bash
# Find feature branches matching task IDs
git branch -a | grep -i "task-id-pattern"

# Get commits mentioning the task ID
git log --all --oneline --grep="TASK-ID"

# Check what's on staging vs production
git log main..staging --oneline 2>/dev/null || git log master..staging --oneline
```

### 3. Determine deployment status

For each task, determine where the code is:
- **In production** — commits are on the production branch and have been deployed
- **On staging** — commits are on the staging branch but not yet on production
- **In progress** — branch exists but hasn't been merged to staging
- **Not started** — no branch or commits found

### 4. Write the recap to Notion

Write the recap to the Notion page. Structure:

```markdown
# Sprint Recap — [Sprint Name] ([Date Range])

**Staging URL:** [staging URL if applicable]
**Branch:** `staging`
**Status:** [Summary — N of M tasks complete]

## Summary
[Brief summary: what the sprint focused on, how many tasks completed vs in progress]

## Completed tasks (N of M)

### 1. TASK-ID — Task title
**Status:** Done — on staging / in production
**Branch:** `feature/task-description`
**URL:** [staging URL if applicable]

[Description of what was built — pull from the Notion card + git commits]

**What to test:**
- [Specific testing checklist items]

---
[Repeat for each completed task]

## In progress (N of M)

### N. TASK-ID — Task title
**Status:** In progress — [reason/blocker if any]
**Branch:** [branch name or "None yet"]

[Description of current state]

## Infrastructure improvements
[List any cross-cutting infra changes, if any]

## Before production rollout
[Checklist of items needed before merging to production]

## Git branch summary
[Table of branches, their base, status, and merge target]
```

### 5. Create a GitHub PR with release notes

After writing the Notion recap, create a PR from the staging branch to the production branch using `gh pr create`. Adapt the recap content into a PR-appropriate format:

```bash
gh pr create --base main --head staging \
  --title "[Sprint Name] — Release Notes ([Date Range])" \
  --body "$(cat <<'EOF'
## Summary
[Condensed version of the recap — feature list, breaking changes, infrastructure]

## Completed Features
[Each feature as a subsection with brief description]

## Breaking Changes
[Call out anything that could affect existing users or API consumers]

## Infrastructure Improvements
[Bullet list]

## Before Merging
- [ ] [Pre-merge checklist items]

## Test Plan
- [ ] [Specific test steps for reviewers]

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Include commit count and file change stats in the PR summary:
```bash
git log main..staging --oneline | wc -l
git diff main..staging --stat
```

If a PR already exists from staging to the production branch, inform the user and ask whether to update it or skip.

### 6. Key guidelines

- For each task, include a **"What to test"** section with specific, actionable checklist items
- Note any **breaking changes** that need advance notice before production rollout
- Call out tasks that are **blocked** and what they're waiting on
- Include **staging URLs** for every feature that has a UI component
- If a task has been partially tested, note what's been verified and what hasn't
- Cross-reference related tasks when they overlap
- The recap should be useful for both the dev team AND for sending to stakeholders as a testing guide
- The PR description should be scannable by reviewers — use tables and checklists, not long paragraphs
- Update the Notion task cards' status if they're out of date based on what git shows
