# claude-standards

Shared Claude Code configuration for all Standard Co. projects. Projects inherit a base and layer their own overrides on top.

## Set it up

Open a Claude Code session in the project you want to configure — Opus or better — and paste this:

```
Set this project up to use https://github.com/standardco/claude-standards.
Read its ONBOARDING.md and follow it.
```

Your session will ask one question up front — whether you want the skills in **just this project** or in **all your projects** on this machine — and handle the rest: secret scanning, cloning, wiring the base rules, installing the skills and review agents, and verifying that it all actually took effect.

You don't need to read anything first, and you don't need to be at Standard Co. — the repo is public and the setup works for any project.

## What's in here

| Path | Purpose |
|------|---------|
| `CLAUDE.md` | Base instructions every project inherits |
| `ONBOARDING.md` | Bootstrap for a developer setting this up for the first time |
| `.mcp.json` | Shared MCP server definitions |
| `.pre-commit-config.yaml` | Secret scanning — the hook this repo requires and runs on itself |
| `.claude/settings.json` | Default Claude settings (non-secret) |
| `.claude/agents/` | Parallel code-review subagents |
| `.claude/commands/` | Shared slash commands |
| `.claude/skills/` | Reusable task templates |
| `docs/` | Runbooks: secrets, privacy, MCP catalog, adoption guide |
| `examples/` | Starter templates for project-level config |

## Once you have it

The setup above installs `/adopt-standards`. After that, adopting another project is one command:

```
/adopt-standards adopt      # full setup for a new or existing project
/adopt-standards resync     # pull in changes after this repo is updated
/adopt-standards verify     # read-only check that a setup still works
```

Run `verify` on any project you haven't touched in a while. Two parts of the setup fail silently — an unresolved `CLAUDE.md` import and skills that were never installed — and both leave a project looking configured with none of the rules in effect.

[`docs/adoption.md`](docs/adoption.md) is the explainer: what the pieces are, how they fit, and which decisions you have to make. The steps live in the skill so they exist in one place.

## Onboarding a developer

Point them at this repo and the paste-in above — that works for anyone, inside the org or outside it, and needs nothing from you.

Standard Co. teams have a second option: an onboarding share link that opens directly in a Claude Code session, paired with an internal page carrying org-specific context. Ask Claude to **refresh the onboarding link**, then send the URL. It's a convenience, not a requirement — the repo alone is enough.

The link is your org's guide, not a per-person invite:

- **Same URL for everyone.** Don't generate a new one per developer.
- **Refreshing updates in place.** Anyone you sent it to previously gets the new content — no need to resend.
- **Only create a second link** for a genuinely different guide, such as a narrower contractor version.

Say *"update the existing onboarding guide"* rather than *"create an onboarding link"* — the second phrasing can mint a duplicate. You don't need to keep the URL to hand; Claude can look it up.

`ONBOARDING.md` is deliberately thin. Everything that changes often — the procedure, the modes, the checks — lives in `/adopt-standards`, which developers get by cloning. So the repo can evolve substantially before the link needs refreshing at all.

## Non-negotiables

Adopting this repo doesn't soften these. Full detail in [`docs/data-privacy.md`](docs/data-privacy.md) and [`docs/secrets.md`](docs/secrets.md).

- Never put raw customer data, PII, or credentials into a Claude prompt. De-identify first.
- Development and testing use synthetic or anonymized data. Never production data.
- Secrets come from AWS Secrets Manager or 1Password. Not `.env`, not commits, not chat history.
- Secret scanning is wired at `git init` time, not later.

## Ownership

One person owns `.mcp.json`. New MCP entries require a PR with a brief justification comment. When a vendor ships an official MCP, the in-house wrapper gets deprecated in the same PR.

The repo wins. Don't sync it back to slides or wikis — update here and let downstream sources pull from it.

**Owner:** Standard Code LLC — developer@standardco.de
