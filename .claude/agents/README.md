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

## Adding a new agent

1. Create a markdown file here with YAML front matter (`name`, `description`, `tools`).
2. Write a tight system prompt — one job, no overlap with existing agents.
3. List only the tools the agent needs (default: `Read`, `Grep`, `Glob`).
4. Add a row to the table above.
