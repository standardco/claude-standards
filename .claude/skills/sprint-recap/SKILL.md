---
name: sprint-recap
description: Generate a sprint recap from a Notion sprint board and git history, then write it to a Notion page
when_to_use: Use when the user asks to generate or update a sprint recap
argument-hint: "[notion-sprint-board-url] [notion-recap-page-url]"
allowed-tools: Bash(git log *) Bash(git diff *) Bash(git branch *)  Grep Read
---

# Sprint Recap Generator

Generate a comprehensive sprint recap by cross-referencing a Notion sprint board with git history, then write the results to a Notion page.

## Inputs

Ask the user for anything not provided as arguments:
- **Sprint board URL** — Notion database with the sprint tasks
- **Column/filter** — which column or status group to pull tasks from (e.g. "JAP Upload", "In Progress", "Current Sprint")
- **Recap page URL** — Notion page where the recap should be written
- **Staging URL** — if the project has a staging environment, include it in the recap

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

# Check what's on staging vs main/master
git log main..staging --oneline 2>/dev/null || git log master..staging --oneline
```

### 3. Determine deployment status

For each task, determine where the code is:
- **In production** — commits are on `main`/`master` and have been deployed
- **On staging** — commits are on `staging` branch but not yet on `main`/`master`
- **In progress** — branch exists but hasn't been merged to staging
- **Not started** — no branch or commits found

### 4. Write the recap

Write the recap to the Notion page. Structure:

```markdown
[Table of contents]

## Sprint overview
[Brief summary: what the sprint focused on, how many tasks completed vs in progress, staging URL if applicable]

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

## Git branch summary
[Table of branches, their base, status, and merge target]
```

### 5. Key guidelines

- For each task, include a **"What to test"** section with specific, actionable checklist items
- Note any **breaking changes** that need advance notice before production rollout
- Call out tasks that are **blocked** and what they're waiting on
- Include **staging URLs** for every feature that has a UI component
- If a task has been partially tested, note what's been verified and what hasn't
- Cross-reference related tasks when they overlap
- The recap should be useful for both the dev team AND for sending to stakeholders as a testing guide
- Update the Notion task cards' status if they're out of date based on what git shows
