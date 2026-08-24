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
- Check for a Claude account connector before wiring a server — no credential to manage where one exists.
- Where you do need a server, its definition lives in that project's `./.mcp.json` — team-shared and version-controlled, not in `.claude/`. This repo ships no server template.
- If you find yourself reaching for raw HTTP twice against the same service, that's a signal to add an MCP.

See [`docs/mcps.md`](docs/mcps.md) for the decision rules and rationale.

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

## Formatting rules

- When composing emails or messages for me to copy-paste, use plain text without markdown blockquote formatting. Use `pbcopy` to place the text directly into the clipboard instead of displaying it in the terminal.

---

## Out of scope (not yet)

These workflows exist but have no repo artifact yet:

- Scheduled tasks (`/schedule`)
- Dispatch routing (phone → desktop)
- AI dashboard monitoring agents
