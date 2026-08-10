---
name: adopt-standards
description: Wire a project to inherit claude-standards — base instructions, review agents, and skills — then verify the standards are actually in effect; also re-syncs and audits an existing setup
when_to_use: Use when setting up a new or existing project to use claude-standards ("adopt the standards", "set this repo up with our Claude config"), when re-syncing after claude-standards changes, or when checking whether a project's setup is actually working
argument-hint: "[adopt | resync | verify]"
allowed-tools: Bash(git clone *) Bash(git submodule *) Bash(git log *) Bash(git rev-parse *) Bash(git status *) Bash(git remote *) Bash(diff *) Bash(cp *) Bash(mkdir *) Bash(ls *) Bash(grep *) Bash(cat *) Bash(gitleaks *) Bash(pre-commit *) Read Write Edit Grep Glob
---

# Adopt claude-standards

Wire a project to inherit our shared standards — base instructions, review agents, and skills — and then prove they're actually loaded.

The proving is the point. Two parts of this setup fail **silently**: a `CLAUDE.md` import that doesn't resolve, and skills that were never installed. In both cases nothing errors, the project looks configured, and none of the standards are in effect. A developer who believes the privacy rules are loaded when they aren't is worse off than one who knows they're unconfigured.

> Rationale for each step lives in [`docs/adoption.md`](../../../docs/adoption.md). This skill is the executable procedure; that doc is the explainer. If they disagree, this file is correct.

## Modes

Take the mode from the argument, or infer it: no `CLAUDE.md` import → `adopt`; import present → `verify` unless the user asked to update.

- **`adopt`** (default) — full setup for a new or existing project
- **`resync`** — pull in changes after `claude-standards` has been updated
- **`verify`** — check an existing setup without changing anything

## Inputs

Infer what you can, ask for the rest. Never guess at these:

- **Where `claude-standards` lives** — submodule at `./claude-standards/`, or a shared clone elsewhere. Determines the import path.
- **Notion URLs and task ID prefix** — unknowable from the repo. A guessed sprint board URL points a skill at the wrong workspace.
- **How credentials are injected** — 1Password CLI or AWS SSM. If unclear, stop and ask. Never hardcode "for now".

Branch names, stack, and test runner you can read from the repo. Do that rather than asking.

---

## Mode: adopt

### 1. Secret scanning, before anything else

First deliberately. Later steps put credential-shaped placeholders into version-controlled files, and the moment those get real values an unprotected repo is one `git add -A` from a leak. The base `CLAUDE.md` says to wire this at `git init` time, not later — so do it before creating the files that carry the risk.

```bash
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.0
    hooks:
      - id: gitleaks
EOF

pre-commit install
gitleaks detect --source . --verbose      # full history, not just the working tree
```

A clean working tree says nothing about history — deleted secrets persist in old commits, which is the case this step exists to catch. On a hit: **rotate first** (assume compromised), then clean history with `git filter-repo` or BFG, force-push, tell the team to re-clone. Do not continue with live leaked credentials.

Check `.gitignore` covers a bare `.env`, not only `.env.local`. The base rules permit `.env` for non-secret local config, so the file will exist, and "one connection string, temporarily" is how it stops being non-secret.

### 2. Place the repo

```bash
git clone https://github.com/standardco/claude-standards.git
```

Submodule at `./claude-standards/` for per-project version pinning; one shared clone for a developer working across several repos. Settle it now — it determines the import path.

### 3. Import the base CLAUDE.md, then prove it resolved

Add the import as the **first line** of the project's `CLAUDE.md`:

```markdown
@./claude-standards/CLAUDE.md
```

If the project already has a `CLAUDE.md`, insert above the existing content. Never overwrite it.

A bad path produces no error and no base rules — but **you cannot verify it from here.**

Two reasons, both absolute:

- If you created or edited `CLAUDE.md` in this session, the import was not present when the session started, and imports resolve at session start. There is nothing loaded to check.
- You read the base file with `Read` to do this setup. You can answer questions about the base rules from that, whether or not the import ever resolves. Any self-check is contaminated.

So: write the import, state plainly that it is **unverified**, and defer the check to step 10. Never report the import as working on the strength of your own answer.

**If the import points outside the project directory, warn them about the trust prompt before they meet it.** Claude Code asks on the next session start — *"Allow external CLAUDE.md file imports? … Never allow this for third-party repositories."* They must answer **yes**; it's their own standards repo at a controlled path, not the third-party case the warning targets. Answering no leaves the base rules unloaded while the project still looks configured, and nothing later flags it. A submodule inside the project raises no prompt at all.

**Stop here if it didn't resolve.** Everything downstream assumes the base rules are live.

### 4. Install skills

**The import does not bring skills.** It inlines instruction text; skills are discovered from directories. Wiring the import and typing `/sprint-recap` does nothing, which reads as "the repo is broken".

**Ask where they want it, in plain terms** — this is a real choice and it is the first thing a new user gets asked:

> Do you want these skills available in **just this project**, or in **all your projects** on this machine?

- **All your projects** → `~/.claude/skills/`. What most people want.
- **Just this project** → `<project>/.claude/skills/`. Nothing outside this folder changes.

```bash
mkdir -p <skills-dir> && cp -r <standards>/.claude/skills/* <skills-dir>/
```

Don't explain "user level versus project level", `~/.claude` paths, or precedence to get the answer — the two options above are the whole decision, and it's reversible either way. Expand only if asked.

Check for same-named skills before copying; `cp -r` overwrites silently. If both locations hold the same skill, the project copy wins.

Installing everywhere works because the skills are generic — the workflow lives in the skill, per-project values come from `## Skill Configuration`.

**Confirm one actually appears** before moving on: have them type `/` and look for `adopt-standards`. Skills are normally live as soon as the files exist, no restart needed. If it's missing, restart the session; if still missing, the files are in the wrong place.

These are copies. They do not track the source. That's what `resync` is for.

### 5. Install review agents

```bash
mkdir -p .claude/agents && cp -r <standards>/.claude/agents/* .claude/agents/
```

`bug-hunter`, `security-auditor`, `style-enforcer` run **in parallel**, one job each. Don't collapse them into a single reviewer.

### 6. Merge settings

Merge into any existing `.claude/settings.json` rather than copying over it. Add what the stack needs — `Bash(pytest:*)`, `Bash(rspec:*)`, `Bash(yarn:*)`.

`:*` only matches at the **end** of a permission pattern. Mid-pattern it silently never matches, which reads as a rule that does nothing rather than as an error.

### 7. Write Skill Configuration

**Do not ask which skills the project uses.** Every installed skill is available regardless, and an unconfigured skill simply asks at runtime — so there is nothing to gate. Enumerating skills for the user to pick from is the one question here that grows every time a skill is added to `claude-standards`, and it reads as "choose what you get" when it only means "choose what to pre-fill." Write the section and move on.

Add a `## Skill Configuration` section to the project `CLAUDE.md`, using [`examples/project-CLAUDE.md`](../../../examples/project-CLAUDE.md) for the shape. Include an entry for every shared skill that **takes** configuration — you know which those are by reading the skills, not by asking.

Fill in what the repo tells you: branch names, stack, test runner, output paths, clipboard command. Stub what it can't, with a visible marker:

```markdown
### sprint-recap
- **Sprint board:** <TODO — fill in when you first run /sprint-recap>
- **Staging branch:** `staging`
- **Production branch:** `main`
- **Task ID prefix:** `<TODO>`
```

A stub is not a failure state. The skill prompts for anything missing the first time it runs, which is the moment the user actually knows the answer. Guessing a Notion URL to avoid a stub is strictly worse.

List the stubbed values in the hand-back summary (step 10) so nothing is silently left blank.

### 8. MCP servers — the leak step

```bash
cp <standards>/.mcp.json ./.mcp.json
```

Remove unused servers, then swap placeholders. **This is where credentials leak.** Every server carries a bare `<GITHUB_PAT>`-style placeholder in an `env` block that takes a literal string, and `.mcp.json` is version-controlled and team-shared by design.

Never write a real credential into it. Source of truth is AWS Secrets Manager and 1Password; values are injected at runtime via the 1Password CLI or AWS SSM. If you don't know how a given value is injected, stop and ask — see [`docs/secrets.md`](../../../docs/secrets.md).

`.mcp.json` belongs at the project root, not in `.claude/`.

### 9. Document de-identification

Every project `CLAUDE.md` needs a `## De-identification` section — required, not optional. Which fields are removed, which are transformed and how, where synthetic test data comes from. Template in [`docs/data-privacy.md`](../../../docs/data-privacy.md).

If the user doesn't know yet, leave it stubbed and flag it. A wrong process documented as correct is worse than an obvious gap.

### 10. Verify and hand back

Run the `verify` mode checklist below, then report plainly: what is set up, what was skipped and why, what still needs a human. Credential injection and de-identification are the two that usually do.

End with a short **"what still needs you"** list — every `<TODO>` stub written in step 7, plus anything deferred. That list is what replaces asking up front: the user leaves with a working setup and a known set of blanks, rather than having answered a questionnaire before seeing any of it work.

On a project with existing code, recommend a baseline security audit (`/security-audit full`) before new work starts — it surfaces pre-existing vulnerabilities and gives the team a ranked fix list. Recommend it; don't run it unasked, since it's a substantial pass.

---

## Mode: resync

Installed skills and agents are copies, so they drift. Show what changed before touching anything.

```bash
git -C <standards> log --oneline -20              # what moved upstream
git -C <standards> pull
diff -rq ~/.claude/skills/ <standards>/.claude/skills/
diff -rq .claude/agents/ <standards>/.claude/agents/
```

Sort the differences into three buckets and treat them differently:

- **Upstream-only changes** — new or updated skills. Copy them in.
- **Local modifications** — a deliberately customised project-local skill. Do **not** overwrite. Show the diff and let the user decide.
- **Local-only files** — a project's own skills. Leave them alone.

`.mcp.json` never auto-syncs. If new servers appeared upstream, name them and let the user pull them in with the right credential wiring.

If the project imports `CLAUDE.md` rather than copying it, base-rule changes are already live — say so rather than implying action is needed. If it copied, diff against the recorded commit SHA.

---

## Mode: verify

Read-only. Check each item by running it, not by looking for the file.

**First, establish which repo you're in.** Running this against `claude-standards` itself is a different check than running it against a project that adopts it, and reporting the wrong one sends people to "fix" things that are correct.

```bash
git remote -v | grep -q 'claude-standards' && echo "source repo"
```

### In an adopting project

- [ ] **Import check — requires a fresh session.** See below; do not run it in a session that performed the setup.
- [ ] At least one skill appears and runs — invoking this skill is itself proof
- [ ] `.claude/agents/` has all three reviewers
- [ ] `pre-commit run --all-files` passes
- [ ] `gitleaks detect` clean on full history
- [ ] `## De-identification` and `## Skill Configuration` present in the project `CLAUDE.md`
- [ ] `git grep -nE '<[A-Z_]{2,}>' -- '*.json' '*.md'` returns nothing committed

### In `claude-standards` itself

Three checks don't apply, and reporting them as failures is wrong:

- **No import to verify.** The repo *is* the base; its `CLAUDE.md` loads directly.
- **No `## De-identification` or `## Skill Configuration`.** Those are what adopting projects write. The base defines the requirement; it doesn't consume it, and it holds no project data.
- **Placeholders are intentional.** `.mcp.json`, `examples/`, and the docs are templates. Unresolved `<PLACEHOLDER>` strings are correct here and nowhere else.

Everything else applies, and the secret-scanning checks matter **more** here, not less — this is the repo people clone as their starting point, so a gap propagates to every project downstream. Check them explicitly rather than waving them through as self-evidently fine.

### The import check — fresh session only

Imports resolve at session start, and a session that ran the setup has read the base file directly. Both make self-verification impossible: a session can create a broken import and still answer questions about the base rules perfectly.

Tell the user to quit and reopen Claude Code in the project directory. If the import is external, the *"Allow external CLAUDE.md file imports?"* prompt appears on start — they answer **yes**, or nothing loads. Then ask:

> What does my CLAUDE.md say about when to use MCP instead of raw HTTP?

**Resolved:** answers from the base rules — MCP by default for anything used more than once by two or more devs, official vendor MCP where one exists, raw HTTP only for one-shots and public unauthenticated APIs, `./.mcp.json` at the repo root.

**Not resolved:** says it doesn't know, or goes reading files. The project has none of the base rules despite looking configured.

Not verification: asking the setup session, reading the import line back out of `CLAUDE.md`, or `/memory` — which is a picker for editing memory files, not a list of what loaded. Avoid "where do secrets live?" as the question; a model may answer 1Password from general knowledge regardless.

Relative imports above the project root (`@../`, required by the shared-clone layout) are the likeliest to fail and the reason this check exists.

### Reporting

Say what failed and what it means in practice — "the import didn't resolve, so no privacy or secrets rules are loaded" is the finding, not "step 3 incomplete". Mark inapplicable checks as not applicable **with the reason**; a bare "skipped" reads as an untested gap.

## Key guidelines

- **Verify, don't assume.** The two silent failures are the reason this skill exists. A file being present is not evidence a rule is loaded.
- **Merge, never overwrite.** Existing `CLAUDE.md`, settings, and customised skills represent decisions someone made.
- **Ask for what the repo can't tell you.** Notion URLs, task prefixes, credential injection. Read branch names and stack yourself.
- **Ask about decisions, never about inventory.** A question the user has to answer once is fine — import path, install scope, MCP servers. A question that lists things from this repo and asks them to tick boxes is not a decision, it's data entry, and it gets longer every time `claude-standards` grows. Stub instead, and say what you stubbed.
- **Stop at the credential step if unsure.** Every other step here is reversible. That one isn't.
- **Report gaps as gaps.** A stubbed de-identification section flagged out loud beats an invented one.

## What not to do

- Don't run end to end silently — several steps need the user's input
- Don't hardcode a credential in `.mcp.json`, even temporarily
- Don't overwrite a project-local skill during `resync` without showing the diff
- Don't continue past a failed import verification
- Don't invent a de-identification process, Notion URL, or task ID prefix
- Don't report success on setup you didn't verify
