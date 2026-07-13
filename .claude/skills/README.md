# .claude/skills/

Reusable task templates. Skills are longer-form prompts for multi-step workflows that appear too often to re-type but are too complex for a simple slash command.

## Convention: generic skills, project-specific context

Skills in this repo are **generic** — they define the workflow steps, output format, and guidelines without hardcoding project-specific values (URLs, branch names, Notion page IDs, task ID prefixes, etc.).

Each project provides its own context in its `CLAUDE.md` under a `## Skill Configuration` section. Skills check that section first before asking the user for inputs. This keeps skills reusable across projects while still working out of the box for any given project.

See [`docs/adoption.md`](../../docs/adoption.md) for how to set up the `## Skill Configuration` section in a project's `CLAUDE.md`, and [`examples/project-CLAUDE.md`](../../examples/project-CLAUDE.md) for a template.

## Adding a skill

1. Create a `<skill-name>/SKILL.md` file here.
2. Write a structured prompt with clear inputs, steps, and expected outputs.
3. Keep it generic — no project-specific URLs, IDs, or branch names. Use the project's `CLAUDE.md` for that.
4. Document it in this README.

## Current skills

| Skill | Trigger | What it does |
|-------|---------|-------------|
| `/sprint-recap` | "generate a sprint recap" | Cross-references Notion sprint board with git history, writes recap to Notion, creates a GitHub PR with release notes |
| `/user-docs` | "create user documentation" | Reads views, controllers, and routes to generate end-user feature guides with screenshot placeholders |

### sprint-recap

Gathers task cards from a Notion sprint board, correlates with git branches and commits, determines deployment status, writes a structured recap to a Notion page, and creates a GitHub PR from staging to production with the recap as release notes.

**Usage:** `/sprint-recap [sprint-board-url] [recap-page-url]`

**Project context needed** (in `CLAUDE.md` → `## Skill Configuration`):
- Sprint board Notion URL
- Recap page Notion URL or ID
- Staging and production branch names
- Staging environment URL
- Task ID prefix (e.g. `PROJ-`, `EDU-`)

### user-docs

Reads the actual codebase to produce accurate user documentation — every field name, button label, and column header comes from the view templates. Outputs markdown with `[SCREENSHOT: ...]` placeholders suitable for GitHub, SharePoint wiki, or Notion.

**Usage:** `/user-docs [feature name or 'all new features']`

**Project context needed** (in `CLAUDE.md` → `## Skill Configuration`):
- Output path (defaults to `docs/user-guide.md`)
- Target audience (admin users, end users, API consumers)
- Notion page URL (optional, for writing docs directly to Notion)
