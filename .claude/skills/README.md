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
| `/adopt-standards` | "set this repo up with our Claude config" | Wires a project to inherit claude-standards, verifies the rules are actually loaded, re-syncs after upstream changes |
| `/sprint-recap` | "generate a sprint recap" | Cross-references Notion sprint board with git history, writes recap to Notion, creates a GitHub PR with release notes |
| `/release-notes` | "write release notes for v2.4.0" | Reconciles a tag range against merged PRs, linked issues, and sprint cards; creates a draft GitHub Release |
| `/user-docs` | "create user documentation" | Reads views, controllers, and routes to generate end-user feature guides with screenshot placeholders |
| `/security-audit` | "run a security audit" | Runs `security-auditor` in five parallel scoped passes, consolidates findings into a ranked report with proposed fixes |
| `/end-of-day` | "clocking out", "wrapping up for the day" | Sweeps for uncommitted and ephemeral state, writes a durable handoff note outside the repo, names tomorrow's first action |
| `/handoff` | "notes for the other session" | Summarizes this session's work as a briefing for a coupled project's Claude session and copies it to the clipboard |

### sprint-recap

Gathers task cards from a Notion sprint board, correlates with git branches and commits, determines deployment status, writes a structured recap to a Notion page, and creates a GitHub PR from staging to production with the recap as release notes.

**Usage:** `/sprint-recap [sprint-board-url] [recap-page-url]`

**Project context needed** (in `CLAUDE.md` → `## Skill Configuration`):
- Sprint board Notion URL
- Recap page Notion URL or ID
- Staging and production branch names
- Staging environment URL
- Task ID prefix (e.g. `PROJ-`, `EDU-`)

### adopt-standards

Sets up a project to inherit this repo — base instructions, review agents, and skills — and then **proves the standards are loaded** rather than assuming the files landed. Two parts of the setup fail silently: a `CLAUDE.md` import pointing at the wrong path, and skills that were never installed because the import doesn't carry them. Both leave a project looking configured with none of the rules in effect, which is worse than an obviously unconfigured one.

Runs secret scanning *first*, before any step writes files. Merges rather than overwrites, since an existing `CLAUDE.md` or customised skill represents someone's decision. Creates no `.mcp.json` and asks nothing about MCP servers — that step was the only one that could leak a credential into version control, and the only one that blocked on an answer the user often didn't have.

Three modes: `adopt` (full setup), `resync` (pull upstream changes, showing local modifications rather than clobbering them), `verify` (read-only check that a setup still works).

[`ONBOARDING.md`](../../ONBOARDING.md) at the repo root is the bootstrap for developers who don't have the skill yet — it covers the four steps up to installing skills, then hands off to `/adopt-standards`. Share it with `ShareOnboardingGuide`. The procedure is maintained here; the bootstrap deliberately stops short of duplicating it.

**Usage:** `/adopt-standards [adopt | resync | verify]` — defaults to `adopt`.

**Project context needed:** none. This skill is what creates it.

### release-notes

Turns a range of shipped commits into technical release notes and creates a **draft** GitHub Release. Git history defines the scope; merged PRs, linked issues, and Notion sprint cards supply the reasoning; the diff verifies anything stated as fact.

Entries are grouped by what the reader must do — breaking changes first, internal work collapsed to one line — and every entry cites a PR number or SHA so it stays checkable later. Reads the actual diff before calling anything breaking, since a missed breaking change is the one error in release notes that becomes someone else's incident. Scrubs client names, PII, internal hostnames, and contributor emails out of card titles and PR bodies before anything reaches GitHub, because a release is public the moment it's published.

Distinct from `/sprint-recap`: that one is time-boxed, pre-merge, and written for reviewers as a PR body with a test plan. This one is version-boxed, post-merge, and written for whoever reads the Releases page later.

**Usage:** `/release-notes [v2.4.0 | v2.3.0..v2.4.0]` — omit the range to use the last tag through `HEAD`.

**Project context needed** (in `CLAUDE.md` → `## Skill Configuration`):
- Release branch and tag scheme
- Notion sprint board URL and task ID prefix (optional — falls back to GitHub issues and PRs)

### security-audit

Runs the `security-auditor` agent five times in parallel, each pass scoped to one domain (SQL injection, auth/access control, XSS/CSRF, input processing, config/secrets), then consolidates and deduplicates findings into a severity-ranked report. Read-only by default; creates a branch with fixes and a PR only when asked.

The domain list lives here; the vulnerability checklist lives in [`.claude/agents/security-auditor.md`](../agents/security-auditor.md). Add new patterns to the agent, not to this skill.

**Usage:** `/security-audit [full | changed-files-only]`

**Project context needed** (in `CLAUDE.md` → `## Skill Configuration`):
- Framework (auto-detected if not specified)
- Excluded paths (optional)

### user-docs

Reads the actual codebase to produce accurate user documentation — every field name, button label, and column header comes from the view templates. Outputs markdown with `[SCREENSHOT: ...]` placeholders suitable for GitHub, SharePoint wiki, or Notion.

**Usage:** `/user-docs [feature name or 'all new features']`

**Project context needed** (in `CLAUDE.md` → `## Skill Configuration`):
- Output path (defaults to `docs/user-guide.md`)
- Target audience (admin users, end users, API consumers)
- Notion page URL (optional, for writing docs directly to Notion)

### end-of-day

Closes out a working session so tomorrow starts from a known state. Handles the three things that get lost overnight: uncommitted work in the repo, ephemeral state outside it (temp directories, hand-started services, exported env vars), and the reasoning behind non-obvious decisions.

Separates intentional changes from incidental churn — lockfile platform lines, test-runner state files, regenerated schema dumps — so nothing pollutes a diff. **Proposes commits rather than making them**; housekeeping is the worst moment for an unreviewed commit. Writes the handoff note outside the repo so it can't be swept into a `git add -A`, and prefers making fragile setup durable over documenting how to rebuild it.

**Usage:** `/end-of-day [optional extra context]`

**Project context needed** (in `CLAUDE.md` → `## Skill Configuration`):
- Handoff note path (defaults to `~/Documents/<project>-handoff-<date>.md`)
- Baseline verification command and its known-good result
- Known ephemeral resources (local database, queue, background workers)
- Whether committing/pushing is pre-authorised (defaults to no — it asks)

### handoff

Bridges two tightly coupled projects running in separate Claude Code sessions — a service and the app that calls it. Summarizes what the current session built, changed, fixed, or found, and copies it to the clipboard for pasting into the other session.

Written **for the other Claude session, not for a human**: exact endpoint paths, parameter names and types, response bodies, and error strings, so the reader can act without follow-up questions it has no way to ask. Verifies status against git and `gh` rather than recollection, because describing local work as deployed is the failure mode that costs the other project a debugging session. Filters to what's actionable, and scrubs PII, credentials, and internal hostnames before anything reaches the clipboard.

Distinct from `/end-of-day`: that one writes a durable note for tomorrow's human in *this* project; this one writes a transient briefing for *another* project's session, now.

**Usage:** `/handoff [target project]` — the target is optional when the context makes it obvious.

**Project context needed** (in `CLAUDE.md` → `## Skill Configuration`):
- Paired projects and their direction (producer or consumer)
- Deployment URLs per environment
- Clipboard command if not macOS `pbcopy`
