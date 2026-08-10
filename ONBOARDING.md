# Adopting claude-standards

You are setting up a project to inherit shared Claude Code standards — base instructions, review agents, and skills — from the `claude-standards` repo.

This file is a **bootstrap**. It covers the steps needed to get the standards on disk and verified, then hands off to the `/adopt-standards` skill, which is the maintained procedure for everything else. Don't reimplement the later steps from memory — install the skill and run it.

Work through this with the developer. Don't run it silently: Step 0 and Step 3 both need their decision, and Step 3 is a hard stop if verification fails.

---

## Step 0 — Ask where this should be installed

**Ask this first, before installing anything, and ask it in plain terms:**

> Do you want these skills available in **just this project**, or in **all your projects** on this machine?

- **All your projects** — most people want this. Installs to `~/.claude/skills/`. Every repo you open gets them, including ones you haven't created yet.
- **Just this project** — installs to `<this-project>/.claude/skills/`. Nothing outside this folder changes. Choose this if you're trying it out first, or if you'll need to customise a skill for this repo specifically.

Either way it's a copy of some files, so it's easy to change later. If they pick "just this project" and want it everywhere afterwards, re-run this step — nothing needs undoing.

Don't explain "user level versus project level", `~/.claude` paths, or precedence rules to make this decision. The two options above are the whole choice. Detail only if they ask.

Use their answer as `<skills-dir>` for the rest of this guide.

---

## Step 1 — Secret scanning, first

Before anything creates credential-shaped placeholders in version-controlled files. The base `CLAUDE.md` says wire this at `git init` time, not later.

```bash
brew install gitleaks pre-commit     # or the platform equivalent

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

A clean working tree says nothing about history — deleted secrets persist in old commits, which is the case this catches. On a hit: **rotate first** (assume compromised), clean history with `git filter-repo` or BFG, force-push, tell the team to re-clone. Do not continue with live leaked credentials.

## Step 2 — Clone the repo

```bash
git clone https://github.com/standardco/claude-standards.git
```

Ask where it should live: a submodule at `./claude-standards/` (version-pinned per project) or one shared clone for all their repos (simpler to update). The answer sets the import path in step 3.

## Step 3 — Import the base rules, then prove it worked

Add this as the **first line** of the project's `CLAUDE.md`, pointing wherever the repo actually landed:

```markdown
@./claude-standards/CLAUDE.md
```

If a `CLAUDE.md` already exists, insert above the existing content. Never overwrite it.

**Now verify, and stop if it fails.** An import pointing at a missing path produces no error, no warning, and no base rules — the developer will believe our privacy and secrets policies are in effect when nothing is loaded. That is the worst outcome of this whole setup.

**This session cannot verify its own work.** Two reasons, and both are absolute:

- `CLAUDE.md` was created *after* this session started. Imports resolve at session start, so there is nothing loaded here to check.
- This session has read the base file with the `Read` tool in order to do the setup. It can answer questions about the base rules from that memory whether or not the import ever resolves.

So verification moves to **Step 6**, in a fresh session. Do not attempt it here, and do not report the import as working. Write the import, tell the user it is unverified, and continue.

`/memory` will not help either — it's a picker for editing memory files, not a list of what loaded. Seeing the import line in `CLAUDE.md` only proves the line exists, not that the path resolved.

## Step 4 — Install the skills

**The import does not bring skills.** It inlines instruction text only; skills are discovered from directories. Wiring the import and then typing `/sprint-recap` does nothing, which reads as a broken repo.

Install to whichever location they chose in Step 0:

```bash
# "All my projects"
mkdir -p ~/.claude/skills && cp -r claude-standards/.claude/skills/* ~/.claude/skills/

# "Just this project"
mkdir -p .claude/skills && cp -r claude-standards/.claude/skills/* .claude/skills/
```

Check first whether anything with the same name is already there, and say so before overwriting — `cp -r` replaces silently.

Installing everywhere works because the skills are generic: the workflow lives in the skill, and per-project values come from a `## Skill Configuration` section in each project's `CLAUDE.md`. If both locations end up holding the same skill, the project copy wins.

**Then confirm a skill actually appears.** Ask them to type `/` and look for `adopt-standards` in the list. Skills are normally picked up as soon as the files exist — no restart. If it isn't there, have them restart the session and check again; if it's still missing, the files went to the wrong place.

These are copies and do not track the source. `/adopt-standards resync` updates them later.

---

## Step 5 — Hand off to the skill

The rest — review agents, settings, `## Skill Configuration`, `.mcp.json` and its credential hazards, de-identification, and the final verification pass — is maintained in the `/adopt-standards` skill you just installed.

Run it:

```
/adopt-standards adopt
```

It will pick up where this left off, ask for the values it can't read from the repo, and finish with a verification checklist. Later, `/adopt-standards resync` pulls in upstream changes and `/adopt-standards verify` re-checks that everything is still actually in effect.

---

## Step 6 — Verify the import, in a fresh session

**This is the only step that cannot happen in the session doing the setup**, for the reasons in Step 3. It needs a session that started *after* `CLAUDE.md` existed and has never read the base file.

Have the user quit and reopen Claude Code in the project directory, then ask:

> What does my CLAUDE.md say about when to use MCP instead of raw HTTP?

**Resolved:** it answers from the base rules — default to MCP for anything used more than once by two or more devs, use the vendor's official MCP where one exists, skip it only for one-shot inspections or public unauthenticated APIs, and server definitions belong in `./.mcp.json` at the repo root.

**Not resolved:** it says it doesn't know, or starts reading files to find out. The project has none of the base rules — no privacy policy, no secrets policy — despite looking configured. Fix the import path and check again.

Two things that do *not* count as verification: asking the setup session (it read the base file already), and reading the import line back out of `CLAUDE.md` (that proves the line exists, not that it resolved).

Relative imports pointing above the project root — `@../claude-standards/CLAUDE.md`, which the shared-clone layout requires — are the case most likely to fail. If the project uses one, this step is not optional.

---

## The parts that aren't optional

Tooling is convenience. These are org policy, and adopting the repo doesn't soften them:

- **Never** put raw customer data, PII, or credentials into a Claude prompt. De-identify first.
- Development and testing use synthetic or anonymized data. Never production data, not even in a throwaway script.
- If you're unsure whether something is PII, treat it as PII.
- Secrets come from AWS Secrets Manager or 1Password. Not `.env`, not commits, not comments, not chat history.

Full detail in `docs/data-privacy.md` and `docs/secrets.md`.
