# Standard Co. — Base Claude Instructions

This file is inherited by all projects. Project-level `CLAUDE.md` files extend it; they do not replace it. Rules here are non-negotiable unless explicitly overridden in the project file with a documented reason.

---

## Data privacy (non-negotiable)

We work with public health and education data. The following rules apply everywhere, always.

- **Never** paste raw customer data, PII, or credentials into a Claude prompt or chat window.
- De-identify before any data enters a Claude workflow. Each project's `CLAUDE.md` must document its specific de-identification process.
- Development and testing use synthetic or anonymized datasets only. Never prod data, even in a throwaway script.
- If you're unsure whether something is PII, treat it as PII.

See [`docs/data-privacy.md`](docs/data-privacy.md) for de-identification rules and patterns.

---

## Secrets

- Source of truth: **AWS Secrets Manager** and **1Password**. Nowhere else.
- `.env` is acceptable for local non-secret config (ports, feature flags). Never use it for real credentials.
- No secrets in Claude prompts, chat history, commits, or comments.
- Pre-commit hooks scan for leaked secrets. Set this up at `git init` time — not later.
- Scanners catch credentials with a recognisable shape, not opaque ones — a storage key or a `secret_key_base` passes clean in a plain `.env`. So credential-bearing paths are declared in the project's `.gitleaks.toml` and enforced there. `.gitignore` reduces accidents; it is not the control.
- Credentials are injected at runtime via the 1Password CLI and AWS SSM. They are never stored in the repo or on disk.

See [`docs/secrets.md`](docs/secrets.md) for the full runbook.

---

## MCP defaults

- Default to MCP for any tool used more than once by two or more devs.
- If the vendor ships an official MCP (GitHub, Notion, Slack, Linear), use it instead of raw HTTP.
- Skip MCP only when: it's a one-shot dev inspection, the API is public and unauthenticated, or it's a secret store (below). *"No server exists yet"* counts only for a third-party API — for one of ours, writing one is the answer.
- Check for a Claude account connector before wiring a server — no credential to manage where one exists.
- Where you do need a server, its definition lives in that project's `./.mcp.json` — team-shared and version-controlled, not in `.claude/`. This repo ships no server template.
- If you find yourself reaching for raw HTTP twice against the same service, that's a signal to add an MCP.
- **Secret stores are never an MCP, official or not.** A tool call returns its result into the model's context, so a 1Password MCP delivers the value we're keeping out. Use [`/1password`](.claude/skills/1password/SKILL.md), which hands values only to a child process.
- **Servers we write are read-only by default.** A write capability is a separate decision, not a later commit. Re-audit one whenever the *upstream* API changes — that is when these break, with no change on our side and nothing raising an error.

See [`docs/mcps.md`](docs/mcps.md) for the decision rules and rationale, and [`/build-mcp-server`](.claude/skills/build-mcp-server/SKILL.md) for writing and maintaining one.

---

## Subagent code review

Three reviewer agents run in parallel on every PR. They are defined in `.claude/agents/`.

| Agent | Focus |
|-------|-------|
| `bug-hunter` | Logic errors, edge cases, broken assumptions |
| `security-auditor` | Vulnerabilities, exposed secrets, injection risks |
| `style-enforcer` | Conventions, patterns, project CLAUDE.md rules |

Do not ask a single agent to do all three jobs. Run them in parallel and consolidate findings before acting.

---

## Dev environment

- Docker for all deployments, local through prod. Same image everywhere.
- Bootstrap target: `git clone && <one command>` should produce a working dev environment.
- Credentials injected at runtime — never stored locally.
- CI/CD runs in the same containers as local development.

---

## Auto-mode usage

Claude Code auto-mode is allowed for:

- Framework migrations across many files
- Boilerplate scaffolding using established team patterns
- Test coverage runs
- Bulk refactors: renames, restructuring, import updates

Always review auto-mode output before merging. Auto-mode does not override the privacy or secrets rules above.

---

## Spec requirements vs. implementation choices

A configured value — a cron expression, a page size, a timeout, a retention window — looks equally tunable whether a stakeholder specified it or someone guessed it. The line of code says nothing either way.

- **Check where a value came from before changing it as an optimisation.** If you can't establish its provenance, that's a question to raise, not a default to assume.
- **Mark spec-derived values at the definition site** — that's where someone about to change one is actually looking. Documenting it elsewhere doesn't reach that moment.
- **When changing a marked value looks warranted, propose it rather than doing it.** Bring the measurement; let a human confirm whether the constraint still holds.
- **Label speculation as speculation** when writing it into a `CLAUDE.md`, a comment, or a handoff note. An unlabelled hypothesis reads as a settled recommendation a fortnight later, and the next session implements it as fact.

```yaml
# SPEC REQUIREMENT — do not change without asking. <where it came from, and what it means>
```

Record confirmed requirements in the project's `CLAUDE.md` — see [`examples/project-CLAUDE.md`](examples/project-CLAUDE.md). List only what you have actually confirmed. A padded list makes guesses look authoritative, which is the failure being prevented.

None of this discourages optimisation. Measure, notice, propose. The check is on provenance, not on change.

---

## Design baseline

Applies to any UI Claude builds or changes — screens, dashboards, forms, reports, HTML email templates, artifacts. A project's own design system, component library or `DESIGN.md` overrides everything here. Match what exists before inventing. When changing existing UI, apply this to what you add, not as grounds for restyling what's already there.

- **Tokens.** Where the project has tokens for colour, spacing and type, use them, and add one rather than hardcode a value none covers.
- **Contrast.** Body text at least 4.5:1, large text at least 3:1. On coloured backgrounds, tint secondary text from that colour rather than using mid-grey; the tint must still pass.
- **States.** Interactive elements show hover and visible focus, a disabled state where they can be disabled, and a loading state where they start async work. Data views have an empty state where the data can be empty, and loading and error states where it's actually fetched. Never simulate latency to show one off. Errors say what went wrong and how to recover.
- **Spacing.** Tight within a group, generous between groups, more space above a heading than below it.
- **Tables.** Quantities right-aligned in tabular numerals. Text, IDs, codes and years left-aligned. Nothing truncated without a way to see it in full.
- **Line length.** Body copy at most about 75 characters per line on wide screens. Check at phone width with realistic-length synthetic content.
- **Labels.** Buttons name their action ("Save changes", not "Submit" or "OK").
- **Motion.** Only to explain a change, and respecting `prefers-reduced-motion`.
- **Icons.** One library at one stroke weight. No emoji as icons.
- **Generic AI defaults.** Avoid these unless the project's design system or the brief asks for them: purple-to-blue gradients, gradient text, cards nested in cards, grids of identical icon-heading-text cards as page structure, small labels above every heading, thick coloured left borders on cards, decorative glass and blur.

See [`docs/design.md`](docs/design.md) for the reasoning and for when a project wants more than a baseline.

---

## Formatting rules

- When composing emails or messages for me to copy-paste, use plain text without markdown blockquote formatting. Use `pbcopy` to place the text directly into the clipboard instead of displaying it in the terminal.

---

## Out of scope (not yet)

These workflows exist but have no repo artifact yet:

- Scheduled tasks (`/schedule`)
- Dispatch routing (phone → desktop)
- AI dashboard monitoring agents
