# Adopting claude-standards in a project

This guide walks through wiring a new (or existing) project to inherit from this repo.

## Three import modes

Choose the mode that fits your project:

| Mode | When to use |
|------|-------------|
| **Base + custom overrides** | Most projects — inherit everything, add project-specific rules |
| **Base + API-specific rules** | Services with complex schemas or endpoint contracts |
| **Base as-is** | Internal tooling with no project-specific Claude rules |

---

## Step 1 — Pull in the base CLAUDE.md

Claude Code supports `@path/to/file` imports in `CLAUDE.md`. Create a project-level `CLAUDE.md` that starts with:

```markdown
@<PATH_OR_URL_TO_CLAUDE_STANDARDS>/CLAUDE.md

# <Project Name> — Project-specific rules

<your overrides here>
```

If you're using a monorepo or have this repo as a git submodule at `./claude-standards/`:

```markdown
@./claude-standards/CLAUDE.md
```

If you're copying files directly (no submodule), copy `CLAUDE.md` to your project root and add a comment at the top noting the source version:

```markdown
<!-- Inherited from claude-standards @ <commit-sha>. Re-sync when base changes. -->
```

---

## Step 2 — Wire the MCP servers

Copy `.mcp.json` to your project root:

```bash
cp claude-standards/.mcp.json ./.mcp.json
```

Then:
1. Remove any servers your project doesn't use.
2. Swap every `<PLACEHOLDER>` for real values — but do not hardcode secrets. Point to 1Password or AWS Secrets Manager references instead.
3. Add any project-specific servers at the bottom.

MCP config lives at `./.mcp.json` (project root), not inside `.claude/`. This keeps it version-controlled and team-shared.

---

## Step 3 — Configure the review agents

Copy the agents directory:

```bash
cp -r claude-standards/.claude/agents/ ./.claude/agents/
```

The `style-enforcer` agent references `CLAUDE.md` for project conventions. No changes needed unless you want to add project-specific review rules — in which case, edit `.claude/agents/style-enforcer.md` directly.

---

## Step 4 — Copy settings

```bash
cp claude-standards/.claude/settings.json ./.claude/settings.json
```

Adjust the `permissions` block for your project's needs. Common additions:

```json
"allow": [
  "Bash(pytest:*)",
  "Bash(rspec:*)",
  "Bash(yarn:*)"
]
```

---

## Step 5 — Set up pre-commit secret scanning

```bash
brew install gitleaks pre-commit

# In your project root
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
EOF

pre-commit install
```

This must be done before the first commit that could contain credentials.

---

## Step 6 — Document your de-identification process

Add a `## De-identification` section to your project's `CLAUDE.md` following the template in [`docs/data-privacy.md`](data-privacy.md). Minimum required:

- What fields are removed before data touches Claude
- What fields are transformed and how
- Where synthetic test data comes from

---

## Step 7 — Swap placeholders

Search your project for `<PLACEHOLDER>` strings and replace them:

```bash
grep -r '<[A-Z_]*>' . --include="*.json" --include="*.md"
```

Do not commit files that still contain unresolved placeholders (except in this template repo itself).

---

## Keeping in sync

When `claude-standards` is updated:

- If using `@import` syntax: changes take effect automatically.
- If using a submodule: `git submodule update --remote`.
- If using copied files: re-copy and resolve diffs manually. The comment at the top of `CLAUDE.md` should include the commit SHA you last synced from.

`.mcp.json` changes are not auto-synced. When a new MCP is added to the base, the project owner must pull it in manually and swap in the relevant credentials.

---

## Checklist

- [ ] Project `CLAUDE.md` imports or copies base `CLAUDE.md`
- [ ] Project `CLAUDE.md` has a `## De-identification` section
- [ ] `.mcp.json` copied, unused servers removed, placeholders replaced
- [ ] `.claude/agents/` copied
- [ ] `.claude/settings.json` copied and adjusted
- [ ] Pre-commit secret scanning installed and tested
- [ ] No `<PLACEHOLDER>` strings remaining in committed files
- [ ] No secrets in `.env`, code, or comments
