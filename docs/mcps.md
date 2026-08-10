# MCP guidance

This repo has an opinion on **when** to reach for an MCP. It does not ship a server list.

Server definitions are project-specific — which database, which host, which credential — and a shared template of them is worse than nothing: it duplicates connectors your Claude account may already provide, and every duplicate is a credential to store, rotate, and keep out of git. Adopting `claude-standards` no longer installs a `.mcp.json`. Add servers when a project actually needs one.

## Decision rules

- **Default to MCP** for any tool used more than once by two or more devs.
- If the vendor ships an official MCP, use it. Don't maintain a wrapper.
- Skip MCP only when: it doesn't exist yet, it's a one-shot dev inspection, or the API is public with no auth.
- Reaching for raw HTTP twice against the same service is a missing MCP. Add one.
- Server definitions live in `./.mcp.json` at the project root — team-shared and version-controlled, not in `.claude/`.
- One owner per `.mcp.json`. New entries need a PR.

## Check for a connector before wiring a server

Claude accounts can carry connectors for common SaaS tools — Notion, Slack, Linear, Google Drive, Atlassian and others — authenticated through the account rather than a token on your machine. Where one exists and covers what you need, **use it instead of a `.mcp.json` entry.** No credential to manage, nothing to rotate, nothing to leak.

So, in order of preference:

| Preference | Use for |
|---|---|
| **Account connector** | Vendor SaaS, where a connector exists and is authorised |
| **`.mcp.json` server** | What connectors can't reach — databases, internal services, self-hosted tools |
| **Raw HTTP** | One-shots and public unauthenticated APIs only |

Connectors are provisioned per account or per org, so confirm your teammates actually have the same ones before a project depends on one. If they don't, a `.mcp.json` entry is the portable answer.

## Adding a server to a project

1. Check for an account connector first. If one covers it, stop here.
2. Check whether the vendor ships an official MCP. Prefer it over a wrapper.
3. Open a PR adding the definition to that project's `.mcp.json`.
4. **Never commit a credential.** Reference an environment variable and inject at runtime — see [`secrets.md`](secrets.md) for the `op run` pattern.
5. Tag the file's owner for review.
6. If it needs a new kind of secret, document that in `secrets.md`.

## Credentials in `.mcp.json`

`.mcp.json` is version-controlled by design, so it must never hold a value. Reference the variable:

```json
"env": {
  "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
}
```

and launch with the value injected:

```bash
op run --env-file=.env.1password -- claude
```

`.env.1password` holds `op://` references rather than secrets, and is gitignored. Full runbook in [`secrets.md`](secrets.md).

Literal `<PLACEHOLDER>` values are not a mechanism. They leave no working path, and the path of least resistance from there is pasting the real token in — which is how credentials end up committed.

## Replacing a wrapper with an official MCP

When a vendor ships an official server that replaces an in-house wrapper, do it in one PR: add the official entry, remove the wrapper, update any `package.json` that carried it, and tell the affected teams.
