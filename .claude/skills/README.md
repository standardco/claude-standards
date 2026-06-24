# .claude/skills/

Reusable task templates. Skills are longer-form prompts for multi-step workflows that appear too often to re-type but are too complex for a simple slash command.

## Adding a skill

1. Create a `<skill-name>/SKILL.md` file here.
2. Write a structured prompt with clear inputs, steps, and expected outputs.
3. Document it in this README.

## Current skills

| Skill | Trigger | What it does |
|-------|---------|-------------|
| `/sprint-recap` | "generate a sprint recap" | Cross-references Notion sprint board with git history, writes recap to Notion, creates a GitHub PR with release notes |
| `/user-docs` | "create user documentation" | Reads views, controllers, and routes to generate end-user feature guides with screenshot placeholders |

### sprint-recap

Gathers task cards from a Notion sprint board, correlates with git branches and commits, determines deployment status, and writes a structured recap to a Notion page. Optionally creates a PR with the recap as release notes.

**Usage:** `/sprint-recap [sprint-board-url] [recap-page-url]`

### user-docs

Reads the actual codebase to produce accurate user documentation — every field name, button label, and column header comes from the view templates. Outputs markdown with `[SCREENSHOT: ...]` placeholders suitable for GitHub, SharePoint wiki, or Notion.

**Usage:** `/user-docs [feature name or 'all new features']`
