# claude-standards

Shared Claude Code configuration for all Standard Co. projects. Projects import a base and layer their own overrides on top.

## What's in here

| Path | Purpose |
|------|---------|
| `CLAUDE.md` | Base instructions every project inherits |
| `.mcp.json` | Shared MCP server definitions |
| `.claude/settings.json` | Default Claude settings (non-secret) |
| `.claude/agents/` | Parallel code-review subagents |
| `.claude/commands/` | Shared slash commands |
| `.claude/skills/` | Reusable task templates |
| `docs/` | Runbooks: secrets, privacy, MCP catalog, adoption guide |
| `examples/` | Starter templates for project-level config |

## How to adopt in a project

See [`docs/adoption.md`](docs/adoption.md) for the full walkthrough. The short version:

1. Copy `CLAUDE.md` into your project root (or symlink it).
2. Add a project-level `CLAUDE.md` that starts with `@claude-standards/CLAUDE.md` to inherit the base.
3. Copy `.mcp.json` and add project-specific servers.
4. Copy `.claude/agents/` if you want parallel PR review.
5. Swap every `<PLACEHOLDER>` for real values.

## Three import modes

- **Base + custom overrides** — inherit everything, add project rules below
- **Base + API-specific rules** — inherit everything, add endpoint / schema context
- **Base as-is** — no overrides, works for internal tooling

## Ownership

One person owns `.mcp.json`. New MCP entries require a PR with a brief justification comment. When a vendor ships an official MCP, the in-house wrapper gets deprecated in the same PR.

The repo wins. Don't sync it back to slides or wikis — update here and let downstream sources pull from it.

**Owner:** `<OWNER_NAME>` — `<OWNER_EMAIL>`
