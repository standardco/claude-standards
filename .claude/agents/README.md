# .claude/agents/

Subagent definitions for parallel PR code review.

Three agents run concurrently on every pull request. Each has a narrow focus and a restricted tool allowlist so they don't step on each other.

| File | Role |
|------|------|
| `bug-hunter.md` | Logic errors, edge cases, broken assumptions |
| `security-auditor.md` | Vulnerabilities, exposed secrets, injection risks |
| `style-enforcer.md` | Conventions, CLAUDE.md compliance, duplication |

## How to trigger a review

```
Run bug-hunter, security-auditor, and style-enforcer in parallel on the diff in this PR.
Consolidate findings before presenting them to me.
```

## Reuse before adding

`security-auditor` is also the engine behind `/security-audit`, which runs it in five parallel domain-scoped passes. When you need broader security coverage, add patterns to `security-auditor.md` — don't define parallel agents inside a skill. One checklist, one file, no drift.

## Adding a new agent

1. Create a markdown file here with YAML front matter (`name`, `description`, `tools`).
2. Write a tight system prompt — one job, no overlap with existing agents.
3. List only the tools the agent needs (default: `Read`, `Grep`, `Glob`).
4. Add a row to the table above.

**A single `.md` file, directly in this directory.** Agents are discovered by filename. A subdirectory, or a file with no front matter, isn't discovered as anything — it sits here looking installed while doing nothing, and nothing reports it.

**Everything in this directory travels.** `/adopt-standards` copies it wholesale into every project that adopts or resyncs:

```bash
cp -r <standards>/.claude/agents/* .claude/agents/
```

So a stray file here lands in every downstream repo. If what you have is a skill, it belongs in [`.claude/skills/`](../skills/); if it's notes, it belongs in `docs/`.
