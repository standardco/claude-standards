# Adopting claude-standards in a project

This is the **explainer**: what the pieces are, how they fit together, and which decisions you have to make. It is not a step-by-step procedure.

The procedure is the [`/adopt-standards`](../.claude/skills/adopt-standards/SKILL.md) skill:

```
/adopt-standards adopt      # full setup for a new or existing project
/adopt-standards resync     # pull in changes after this repo is updated
/adopt-standards verify     # read-only check that a setup still works
```

If you don't have the skill yet, [`ONBOARDING.md`](../ONBOARDING.md) at the repo root bootstraps you to the point of having it, then hands off. It's also what you share with a new developer — see `ShareOnboardingGuide`.

Keeping the steps in one place is deliberate. A procedure duplicated across a doc and a skill drifts, and the copy that drifts is always the one someone follows.

---

## The mental model

Adoption moves three different kinds of thing into your project, and they behave differently. Most confusion about this repo comes from treating them the same.

| What | How it arrives | Updates when this repo changes? |
|------|----------------|---------------------------------|
| Base instructions (`CLAUDE.md`) | **Imported** via `@path` | Yes, automatically — the import resolves at load time |
| Skills (`.claude/skills/`) | **Copied** to `~/.claude/skills/` or the project | No — copies drift, use `resync` |
| Review agents (`.claude/agents/`) | **Copied** to the project | No — copies drift, use `resync` |
| MCP servers (`.mcp.json`) | **Copied**, then credentials wired | No — never auto-syncs, always manual |

The consequence people trip on: **the `@import` does not bring skills.** It inlines instruction text. Skills are discovered from directories. Wiring the import and then typing `/sprint-recap` does nothing, and the natural conclusion is that the repo is broken.

## Two failures that are silent

Both leave a project looking correctly configured with none of the standards in effect. That is worse than an obviously unconfigured project, because nobody goes looking.

**A `CLAUDE.md` import that doesn't resolve.** `@./claude-standards/CLAUDE.md` assumes the repo is at exactly that path. Point it somewhere else and you get no error, no warning, and no base rules — including the privacy and secrets policies.

There is no file or command that reports what loaded. `/memory` is a picker for *editing* memory files, not a list of active instructions, and seeing the import line in `CLAUDE.md` only proves the line exists. **The check has to be behavioural** — ask your session something only the base rules can answer:

> What does my CLAUDE.md say about when to use MCP instead of raw HTTP?

If it can't answer without going and reading files, the import didn't resolve. Relative imports pointing above the project root (`@../`) are the likeliest to fail.

**Skills that were never installed.** Covered above. Confirm a skill actually appears and runs.

`/adopt-standards verify` checks both, plus the rest, by running them rather than looking for files.

## Decisions you have to make

### Where this repo lives

- **Submodule at `./claude-standards/`** — version-pinned per project, updated deliberately with `git submodule update --remote`. Best when different projects need different versions.
- **One shared clone** outside your projects — simpler, one place to pull updates. Best for a single developer working across several repos.

This determines your import path, so settle it before writing the import.

### How much you override

| Mode | When to use |
|------|-------------|
| **Base + custom overrides** | Most projects — inherit everything, add project-specific rules below the import |
| **Base + API-specific rules** | Services with complex schemas or endpoint contracts |
| **Base as-is** | Internal tooling with no project-specific Claude rules |

### Import or copy the base rules

Importing is strongly preferred — updates land automatically. If you must copy `CLAUDE.md` instead, record where it came from so a future re-sync has a baseline to diff against:

```markdown
<!-- Inherited from claude-standards @ <commit-sha>. Re-sync when base changes. -->
```

### Where skills live

Setup asks this as a plain question — *just this project, or all your projects on this machine?* — because that's the whole decision from a user's point of view. The mechanics behind it:

- **All your projects** → `~/.claude/skills/`. The default answer for most people. Works because the skills are generic: the workflow lives in the skill, per-project values come from a `## Skill Configuration` section in each project's `CLAUDE.md`, so one copy serves every repo.
- **Just this project** → `<project>/.claude/skills/`. The right answer when a project needs to **modify a skill's behaviour** rather than just its inputs, or when someone is trying the standards out before committing to them everywhere.

If both exist, the project copy takes precedence. Either way it's a file copy, so switching later costs nothing — don't present it as a decision that needs getting right first time.

## Things worth knowing before you start

**Secret scanning comes first, not last.** Setup writes credential-shaped placeholders into version-controlled files, so the protection has to exist before the values do. The base `CLAUDE.md` puts this at `git init` time. Also audit existing history — deleted secrets persist in old commits, and a clean working tree says nothing about that.

**`.mcp.json` is where credentials leak.** Every server carries a bare `<GITHUB_PAT>`-style placeholder in an `env` block that accepts a literal string, and the file is version-controlled and team-shared by design. Credentials come from AWS Secrets Manager or 1Password and are injected at runtime — never written into the file. If you don't know how a value gets injected, stop and ask. See the [secrets runbook](secrets.md).

**De-identification is required, not optional.** Every project `CLAUDE.md` needs a `## De-identification` section covering which fields are removed, which are transformed and how, and where synthetic test data comes from. Template in [`docs/data-privacy.md`](data-privacy.md). If you don't know the answer yet, stub it and flag it — a wrong process documented as correct is worse than an obvious gap.

**`style-enforcer` reads your `CLAUDE.md`.** The agent needs no changes to work; it picks up project conventions from the file. Edit `.claude/agents/style-enforcer.md` directly only when you want review rules that don't belong in `CLAUDE.md`.

**Permission patterns:** `:*` only matches at the **end** of a pattern. `Bash(git:*)` works; mid-pattern it silently never matches, which reads as a rule that does nothing rather than as an error.

**Skill Configuration is optional, and stubs are fine.** Every installed skill works whether or not it has an entry — an unconfigured skill just asks at runtime, which is usually the moment you actually know the answer. Setup writes the section with `<TODO>` markers rather than interviewing you about skills you may never run. Fill them in as you go.

**Resolve every placeholder before committing.** Check with `git grep -nE '<[A-Z_]{2,}>' -- '*.json' '*.md'`. The exception is this repo itself — `.mcp.json`, `examples/`, and the docs here are templates, so unresolved placeholders are correct in `claude-standards` and nowhere else.

## After adoption

Run a baseline security audit on any project with existing code:

```
/security-audit full
```

This surfaces pre-existing vulnerabilities and gives the team a prioritized fix list before new work starts. See [`.claude/skills/security-audit/SKILL.md`](../.claude/skills/security-audit/SKILL.md).

## Keeping in sync

`/adopt-standards resync` handles this. What it's working around:

- **Imported `CLAUDE.md`** — already current, nothing to do.
- **Submodule** — `git submodule update --remote`.
- **Copied files** — skills and agents are snapshots. `resync` diffs them against upstream and separates genuine upstream changes from your deliberate local modifications, so a customised skill doesn't get silently clobbered.
- **`.mcp.json`** — never auto-syncs. When a new MCP is added to the base, the project owner pulls it in manually and wires the credentials.
