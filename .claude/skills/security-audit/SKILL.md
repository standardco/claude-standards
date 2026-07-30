---
name: security-audit
description: Run a comprehensive security assessment across the codebase using parallel scoped passes of the security-auditor agent, consolidate findings, rank by severity, and propose fixes
when_to_use: Use when the user asks for a security audit, vulnerability assessment, or security review of the codebase
argument-hint: "[scope: full | changed-files-only]"
allowed-tools: Bash(git log *) Bash(git diff *) Bash(git branch *) Bash(git checkout *) Bash(git add *) Bash(git commit *) Bash(gh pr *) Grep Read Glob Edit Write Task
---

# Security Audit

Run a structured security assessment across the codebase by launching the `security-auditor` agent several times in parallel, each pass scoped to a distinct attack surface. Consolidate findings into a ranked report with proposed fixes.

> **Project-specific context** (framework, known risks, excluded paths) should be defined in the project's `CLAUDE.md` under `## Skill Configuration`. Check there first before asking the user.

## Inputs

Check the project's `CLAUDE.md` for defaults, then ask the user for anything still missing:
- **Scope** — full codebase or only changed files (e.g. `git diff main...HEAD`)
- **Framework** — Rails, Django, Express, etc. (auto-detect from Gemfile/package.json/requirements.txt if not specified)
- **Excluded paths** — vendor, node_modules, generated code (excluded by default)

## Steps

### 1. Launch five scoped `security-auditor` passes

Launch the `security-auditor` agent five times in parallel — one pass per domain below.

**Use the existing agent. Do not define new ones and do not restate its checklist in the prompt.** The agent in [`.claude/agents/security-auditor.md`](../../agents/security-auditor.md) already carries what to look for, the severity rubric, and the response format. This skill's job is only to divide the surface area so the passes don't overlap. Keeping the checklist in one file is what stops the skill and the agent from drifting apart.

| Pass | Domain | Where to concentrate |
|------|--------|----------------------|
| 1 | SQL injection and query safety | Models, query objects, raw SQL, migrations, reporting code |
| 2 | Auth and access control | Controllers, middleware, policies, session and token config |
| 3 | XSS, CSRF, and frontend | Views, templates, client-side JS, upload handlers |
| 4 | Input processing and external calls | Parsers, background jobs, webhook handlers, HTTP clients |
| 5 | Config, secrets, and dependencies | Config and env handling, dependency manifests, CI definitions |

Prompt each pass with its domain, the scope, and one instruction: *analyze only your domain — another pass covers the rest.* If a pass spots something outside its domain, it should note it in a single line and move on rather than investigating; step 2 reconciles those. Nothing gets silently discarded.

If a pass returns nothing, that is a valid result. Record it as "no findings" rather than re-running it with a looser prompt.

### 2. Consolidate findings

After all five passes complete:
1. Deduplicate — the same issue found by two passes should appear once
2. Merge related findings — e.g. multiple instances of the same anti-pattern become one finding with multiple locations
3. Assign final severity based on realistic exploitability, not theoretical risk

### 3. Produce the report

Structure the output as a ranked list grouped by severity:

```markdown
## Findings by Severity

### CRITICAL
#### C1. [Short title]
- **File:** `path/to/file.rb:123`
- **Issue:** [One-sentence description]
- **Code:** [Vulnerable snippet]
- **Vector:** [How an attacker exploits this]
- **Fix:** [Minimal code change]

### HIGH
[Same structure]

### MEDIUM
[Same structure]

### LOW
[Same structure]
```

End with a **Remediation Priority** section that groups fixes into:
- **Immediate** — before next deploy
- **This week** — high-severity items
- **This sprint** — medium-severity items
- **Next sprint** — low-severity and hardening

### 4. Propose fixes (only if asked)

Steps 1–3 are read-only. Do not start this step on your own — deliver the report and stop. Write code only when the user asks for fixes after seeing the findings.

When they do:
1. Create a dedicated branch (e.g. `fix/critical-security-issues`) — never commit fixes to `main` or the branch under audit
2. Fix Critical and High items first; confirm before touching Medium or Low
3. Commit with a message referencing the finding IDs
4. Open a PR with a test plan covering the changed behavior

A fix that changes authentication, authorization, or query construction needs a human reviewer regardless of how mechanical it looks.

## Key guidelines

- **Severity = exploitability.** A theoretical attack that requires admin access AND source code access is Low, not Critical. A public endpoint with SQL injection is Critical.
- **Don't flag framework defaults as vulnerabilities.** If Rails escapes output by default, don't flag normal `<%= %>` usage. Flag explicit bypasses like `.html_safe`.
- **Don't pad.** If an audit domain finds nothing, say so. Don't invent findings to fill space.
- **Include the positive.** If the codebase does something well (e.g. parameterized queries in most places), note it briefly — it helps the team understand what patterns to follow.
- **Credentials in git history are always Critical** even if the file has since been removed. The secret was exposed the moment it was committed.
- **Rate outbound HTTP separately from inbound.** SSRF (outbound to attacker-controlled URL) and open redirect (inbound redirect to attacker URL) are different vulnerability classes.
