# MCP catalog

## Decision rules

- **Default to MCP** for any tool used more than once by two or more devs.
- If the vendor ships an official MCP, use it. Don't maintain a wrapper.
- Skip MCP only when: it doesn't exist yet, one-shot dev inspection, or public API with no auth.
- Reach for raw HTTP twice against the same service → that's a missing MCP. Add one.
- MCP definitions live in `./.mcp.json` at project root. One owner per file. New entries need a PR.

---

## Wired today

### Postgres
- **Package:** `@modelcontextprotocol/server-postgres`
- **Auth:** Connection string via 1Password at runtime
- **Use:** Primary relational store for most services

### MySQL
- **Package:** `mcp-server-mysql`
- **Auth:** Host/user/password from AWS Secrets Manager
- **Use:** Legacy services and third-party integrations that require MySQL

### SQL Server
- **Package:** `mcp-server-mssql`
- **Auth:** Host/credentials from AWS Secrets Manager
- **Use:** <SQLSERVER_USE_CASE>

### GitHub
- **Package:** `@modelcontextprotocol/server-github` (official)
- **Auth:** PAT scoped to `repo` + PR read/write, stored in 1Password
- **Use:** PR review, issue lookup, branch management from Claude workflows

### Notion
- **Package:** `@notionhq/notion-mcp-server` (official)
- **Auth:** Integration API key from 1Password
- **Use:** Internal knowledge base, project docs, runbooks

### Slack
- **Package:** `@modelcontextprotocol/server-slack` (official)
- **Auth:** Bot token (`channels:read`, `chat:write`) from 1Password
- **Use:** Notifications, incident updates, Claude-triggered messages

### Linear
- **Package:** `linear-mcp-server`
- **Auth:** API key from 1Password
- **Use:** Issue creation, status updates, sprint queries from Claude workflows

### Helpscout
- **Package:** `<HELPSCOUT_MCP_PACKAGE>` (in-house until vendor ships official)
- **Auth:** App ID + secret from AWS Secrets Manager
- **Use:** Support ticket lookup and triage
- **Note:** Deprecate this wrapper when Helpscout ships an official MCP.

### Staging / webhooks
- **Package:** `<INTERNAL_STAGING_MCP_PACKAGE>` (internal)
- **Auth:** API key from AWS Secrets Manager
- **Use:** Webhook inspection, staging environment queries, integration testing
- **Owner:** Platform team

### Ops
- **Package:** `<INTERNAL_OPS_MCP_PACKAGE>` (internal)
- **Auth:** API key from AWS Secrets Manager
- **Use:** Internal operations tooling
- **Owner:** Platform team

---

## Adding a new MCP

1. Check whether the vendor has an official MCP first.
2. Open a PR adding the server definition to `.mcp.json`.
3. Add a row to this catalog with: package name, auth method, use case, owner.
4. Tag `.mcp.json`'s designated owner for review.
5. Document credential setup in `docs/secrets.md` if it requires a new secret type.

## Deprecating an MCP

When a vendor ships an official MCP to replace an in-house wrapper:

1. Add the official MCP entry to `.mcp.json`.
2. Remove the in-house wrapper entry.
3. Update this catalog.
4. Notify teams in the relevant Slack channel.
5. Remove the wrapper package from any `package.json` files.

All in one PR.
