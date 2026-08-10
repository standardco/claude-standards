# Adopting claude-standards

You are setting up a Standard Co. project to inherit our shared Claude Code standards — base instructions, review agents, and skills — from the `claude-standards` repo.

This file is a **bootstrap**. It covers the four steps needed to get the standards on disk and verified, then hands off to the `/adopt-standards` skill, which is the maintained procedure for everything else. Don't reimplement the later steps from memory — install the skill and run it.

Work through this with the developer. Don't run it silently: step 3 needs their decision, and it's a hard stop if verification fails.

---

## Step 1 — Secret scanning, first

Before anything creates credential-shaped placeholders in version-controlled files. The base `CLAUDE.md` says wire this at `git init` time, not later.

```bash
brew install gitleaks pre-commit     # or the platform equivalent

cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
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

Have them run `/memory` and confirm `claude-standards/CLAUDE.md` is listed. Cross-check by asking the session something only the base rules answer, like where secrets live.

Do not continue until it resolves.

## Step 4 — Install the skills

**The import does not bring skills.** It inlines instruction text only; skills are discovered from directories. Wiring the import and then typing `/sprint-recap` does nothing, which reads as a broken repo.

```bash
mkdir -p ~/.claude/skills
cp -r claude-standards/.claude/skills/* ~/.claude/skills/
```

User level means one copy works across every project — the skills are written generic, taking per-project values from a `## Skill Configuration` section in each project's `CLAUDE.md`. Use project-level `.claude/skills/` only when a project needs to modify a skill's behaviour.

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

## The parts that aren't optional

Tooling is convenience. These are org policy, and adopting the repo doesn't soften them:

- **Never** put raw customer data, PII, or credentials into a Claude prompt. De-identify first.
- Development and testing use synthetic or anonymized data. Never production data, not even in a throwaway script.
- If you're unsure whether something is PII, treat it as PII.
- Secrets come from AWS Secrets Manager or 1Password. Not `.env`, not commits, not comments, not chat history.

Full detail in `docs/data-privacy.md` and `docs/secrets.md`.
