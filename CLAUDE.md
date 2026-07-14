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
- Credentials are injected at runtime via the 1Password CLI and AWS SSM. They are never stored in the repo or on disk.

See [`docs/secrets.md`](docs/secrets.md) for the full runbook.

---

## MCP defaults

- Default to MCP for any tool used more than once by two or more devs.
- If the vendor ships an official MCP (GitHub, Notion, Slack, Linear), use it instead of raw HTTP.
- Skip MCP only when: it doesn't exist yet, it's a one-shot dev inspection, or the API is public and unauthenticated.
- MCP server definitions live in `./.mcp.json` — team-shared and version-controlled. Not in `.claude/`.
- If you find yourself reaching for raw HTTP twice against the same service, that's a signal to add an MCP.

See [`docs/mcps.md`](docs/mcps.md) for the full catalog and rationale.

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

## Formatting rules

- When composing emails or messages for me to copy-paste, use plain text without markdown blockquote formatting. Use `pbcopy` to place the text directly into the clipboard instead of displaying it in the terminal.

---

## Out of scope (not yet)

These workflows exist but have no repo artifact yet:

- Scheduled tasks (`/schedule`)
- Dispatch routing (phone → desktop)
- AI dashboard monitoring agents
