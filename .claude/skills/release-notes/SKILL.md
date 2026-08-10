---
name: release-notes
description: Write technical release notes for a tag range by reconciling git history, merged PRs, linked issues, and sprint cards, then create a draft GitHub Release
when_to_use: Use when the user asks for release notes, a changelog for a version, or notes for a GitHub Release — "write release notes for v2.4.0", "what shipped since the last tag", "draft the release"
argument-hint: "[version or tag range, e.g. v2.4.0 or v2.3.0..v2.4.0]"
allowed-tools: Bash(git log *) Bash(git tag *) Bash(git describe *) Bash(git diff *) Bash(git show *) Bash(git rev-list *) Bash(git rev-parse *) Bash(git merge-base *) Bash(git shortlog *) Bash(git fetch *) Bash(gh release *) Bash(gh pr *) Bash(gh issue *) Bash(gh api *) Bash(wc *) Read Write Grep Glob
---

# Release Notes

Turn a range of shipped commits into release notes a developer can act on, and publish them as a **draft** GitHub Release.

The organising idea: **git history is the spine, everything else is annotation.** Commits are the only complete and verifiable record of what actually shipped — sprint cards describe intent, and intent drifts. So the commit range defines the scope, PRs and cards supply the *why*, and the diff verifies anything you're about to state as fact.

> Not to be confused with [`/sprint-recap`](../sprint-recap/SKILL.md). That one is **time-boxed** (one sprint), **pre-merge**, and written for reviewers — it produces a PR body with a test plan. This one is **version-boxed** (tag to tag), **post-merge**, and written for whoever reads the Releases page six months from now. If both apply, run both; they have different readers.

> **Project-specific context** (release branch, tag scheme, Notion sprint board, task ID prefix) should be defined in the project's `CLAUDE.md` under `## Skill Configuration`. Check there first before asking the user.

## Inputs

Check the project's `CLAUDE.md` for defaults, then infer before asking:

- **Range** — from the argument. Accept a single version (`v2.4.0` → previous tag to that tag), an explicit range (`v2.3.0..v2.4.0`), or nothing (previous tag to `HEAD`). Infer it; ask only when tags are ambiguous.
- **Version number** — for an untagged release. Derive from the existing tag scheme, don't impose one. Propose it, and let the user confirm before it goes on a release.
- **Release branch** — where releases are cut from. Defaults to the repo's default branch.
- **Audience** — technical by default: the reader is a developer integrating against this, or a future maintainer bisecting a regression.

## Steps

### 1. Establish the range

The goal is **coverage, not precision**: every change that ships should end up described in some release note. A change mentioned in two releases is mildly redundant; a change mentioned in none is invisible forever. When the two conflict, favour coverage.

The default range is *since the last release*. Start from published releases rather than tags, because a release is what readers actually last saw — a tag with no notes attached is a gap, not a boundary.

```bash
git fetch --tags --quiet
gh release list --limit 10                  # what has actually been published
git tag --sort=-v:refname | head -10        # semver; use --sort=-creatordate for date-based
git log <prev>..<new> --oneline --no-merges
git rev-list <prev>..<new> --count
```

Check these before trusting the range:

- **Is the previous tag an ancestor?** `git merge-base --is-ancestor <prev> <new>` — if not, the tags are on divergent branches and `<prev>..<new>` will report commits that were never released together.
- **Are there tags with no release?** If a tag between the last published release and this one has no notes attached, the work under it was never described. Extend the range to cover it and say that's what you did.
- **How does this repo merge?** Squash-merge repos give one clean commit per PR, so commits *are* the unit. Merge-commit repos need `--no-merges` for the commit list and `--first-parent` for the shape of what landed on the release branch. Detect it (`git log --merges <prev>..<new> | wc -l`) rather than assuming.

If there is no previous release or tag at all, this is the first one: say so, and either cover the full history or ask for a starting point rather than dumping every commit since `git init`.

### 1a. Handle hotfixes and cherry-picks

Hotfixes break the clean tag-to-tag model, and they're common enough to plan for. A fix cut straight to production may already have its own release notes, or may have gone out with none at all — and if it was cherry-picked, the same change exists twice under different SHAs.

```bash
git cherry <prev> <new>                     # '-' marks commits already present upstream
git log <prev>..<new> --cherry-mark --left-right --oneline
gh release view <hotfix-tag> --json body    # did this already get described?
```

Then:

- **Already released with notes** → include it in a short "Also in this release" line pointing at the hotfix release, rather than writing it up twice at full length. The reader upgrading from two versions back still needs to know it's in there.
- **Already released with no notes** → write it up in full here. This is the gap the coverage rule exists to close.
- **Cherry-picked, so present twice** → describe it once, and cite both SHAs.

### 1b. Flag the range, don't silently resolve it

These notes are a draft that developers read and give feedback on, so an explicit question in the output is cheap and a silent assumption is not. When the range is genuinely ambiguous, state what you chose and why in a line at the top of your response — not buried in the notes themselves:

> Covering `v2.3.0..v2.4.0` (31 commits). Extended back to include `v2.3.1`, a hotfix tag with no release notes attached. Two commits are cherry-picks of `v2.3.1` and are described once.

Then let the user correct it. Never quietly narrow the range to make the notes tidier.

### 2. Map commits to PRs and issues

The commit subject is rarely the whole story. Resolve each commit to its PR, which carries the description, labels, and linked issues:

```bash
# commit → PR (reliable even for squash merges)
gh api repos/{owner}/{repo}/commits/<sha>/pulls --jq '.[] | {number, title, url}'

# or the whole set at once, by merge date
gh pr list --state merged --limit 100 \
  --json number,title,body,labels,author,mergedAt,url

# issues closed by those PRs
gh issue view <n> --json number,title,labels,url
```

Labels are the cheapest reliable signal for categorisation — `breaking`, `bug`, `enhancement`, `internal`. Use them where they exist; fall back to reading the change where they don't.

### 3. Pull sprint cards for the "why"

Extract task IDs from commit subjects and branch names using the project's task ID prefix, then fetch those cards from the Notion sprint board for the intent behind the change — the problem being solved, the decision, the constraint.

If the project has no Notion board configured, skip this step and rely on PR bodies and linked issues. The notes are still complete; they just lose a layer of context. Don't block on it, and don't ask the user to go find a board.

**Cards annotate, they never define scope.** A card marked Done whose commits aren't in the range did not ship in this release. Git decides; the card explains.

### 4. Verify before you classify

Read the actual diff for anything you are about to describe as breaking, removed, or contract-changing. This is the one class of error in release notes that costs other people real time — a missed breaking change turns into someone else's production incident, and a phantom one turns into migration work nobody needed to do.

```bash
git diff <prev>..<new> --stat
git diff <prev>..<new> -- <path/to/routes|schema|migrations|public API>
git show <sha>
```

Things that are breaking whether or not anyone labelled them as such:

- Migrations that drop or rename a column, table, or index
- Removed, renamed, or re-pathed routes and endpoints; changed status codes or error shapes
- Changed function or class signatures in anything another project imports
- Renamed or removed environment variables and config keys
- Changed default values — silently the worst of them, because nothing errors
- Major-version dependency bumps that surface through this project's own API
- Raised minimum runtime, language, or database versions

For each one, record the **exact symptom** the reader will observe and the **exact fix**. "Renamed the status field" is not actionable. "`GET /records/{id}` returns `state` instead of `status`; update consumers before deploying" is.

### 5. Group by what the reader must do

Not by commit type, and not chronologically. A reader scans release notes for one question: *does this affect me, and what do I have to change?*

1. **Breaking changes** — first, always, even if there's only one and it's small
2. **Added** — new features and capabilities
3. **Changed** — behaviour changes that aren't breaking
4. **Fixed** — bugs, with the symptom that's now gone
5. **Security** — separate from Fixed, so it survives a skim
6. **Internal** — refactors, dependency bumps, CI, tests. Collapse into a summary line with a commit count; a reader who wants the detail has the compare link.

Every entry cites its source: PR number, or a short SHA where there's no PR. That citation is what makes the notes checkable later.

### 6. Scrub before it goes near GitHub

A GitHub Release is public the moment it's published, and on a public repo the draft is visible to anyone with read access. Sprint cards and PR bodies are written for an internal audience and routinely carry things that must not be republished.

- **Client and customer identity.** Ticket titles are the usual leak: "Fix duplicate patient records for <District>" names a customer and its sector in one line. Rewrite generically — "Fix duplicate record creation on import".
- **PII in any example.** No real IDs, names, emails, or dates of birth in sample payloads or error messages. Synthetic values, same shape and same edge cases.
- **Internal infrastructure.** Staging hostnames, admin paths, queue names, internal service URLs, bucket names.
- **Credentials.** Never the value. Name where it lives — `1Password → <item>`, `AWS Secrets Manager → <path>`.
- **Contributor emails.** `git shortlog -sne` exposes every committer's email address. Use `-sn` and drop the addresses.

Apply this without being asked. The user should not have to remember that a card title was sensitive.

### 7. Create the draft release

```bash
gh release create <tag> --draft \
  --title "<tag> — <one-line theme>" \
  --notes-file <scratchpad>/release-notes.md
```

Write the notes to a file in the scratchpad directory and pass `--notes-file` rather than `--notes` — it keeps backticks, code blocks, and newlines intact through the shell.

Rules for this step:

- **Draft, always.** Publishing is outward-facing and awkward to walk back. Create the draft, show the user the full text and the release URL, and let them publish.
- **Don't create the tag as a side effect.** If `<tag>` doesn't exist, `gh release create` will make it at the target commit. Say that's what will happen and confirm the target first.
- **Check for an existing release** at that tag (`gh release view <tag>`). If one exists, report it and ask whether to update it — never overwrite silently.
- Add the compare link: `https://github.com/<owner>/<repo>/compare/<prev>...<new>`.

Then display the notes in the response. That review is the only check on this skill's output before it becomes public.

## Output structure

```markdown
## <tag> — <one-line theme>

<Two or three sentences: what this release is for, and who needs to act.
If there are no breaking changes, say so explicitly — it's the first thing
a reader wants to know.>

### ⚠ Breaking changes

**<What changed>** (#<pr>)
- **Symptom:** <what the reader observes if they do nothing>
- **Action:** <exactly what to change, with names copied from the code>

### Added
- <Capability, from the user's point of view rather than the implementation's> (#<pr>)

### Changed
- <Behaviour change and what it was before> (#<pr>)

### Fixed
- <The symptom that is now gone, not the internal cause> (#<pr>)

### Security
- <Issue class and affected versions. No exploit detail.> (#<pr>)

### Internal
<N> commits covering <refactors, dependency updates, CI, test coverage>.

---
**Full changelog:** <compare link>
**Contributors:** <names, no email addresses>
```

Drop sections with nothing in them — an empty "Security" heading reads as a claim.

## Key guidelines

- **Never write an entry from a commit subject alone.** Subjects are written for the author's future self, mid-task. Read the PR body or the diff.
- **Describe the effect, not the implementation.** "Fixed a race condition in the cache layer" tells a reader nothing about whether they were affected. "Duplicate rows no longer appear when two imports run concurrently" does.
- **Coverage beats tidiness.** A change described twice costs a reader a few seconds. A change described nowhere is invisible to whoever goes looking for it later. When in doubt, include it and flag it.
- **The draft is the start of a conversation.** Developers review these and give feedback, so surface uncertainty in the response rather than resolving it silently — an ambiguous range, a change you can't classify, a card that doesn't match its commits.
- **A missing breaking change is the failure mode.** When you're unsure whether something is breaking, read the diff. When you're still unsure, list it and say what you're unsure about — an over-flagged change costs a reader a minute, a missed one costs them an incident.
- **Cite everything.** PR number or short SHA on every entry. Six months later it's the only way to check a claim.
- **Internal work gets one line.** It is real work and it does not belong in a reader's way.
- **Match the existing tag scheme.** If the last five tags are `2024.11.1`, do not propose `v3.0.0`.

## What not to do

- Don't publish. Draft, show, and let the user decide.
- Don't let sprint cards define what shipped — a card marked Done whose code isn't in the range didn't ship
- Don't paste card titles, PR bodies, or log output into the notes without scrubbing them first
- Don't dump `git log` as the release notes; a raw commit list is what this skill exists to replace
- Don't invent a version number or a tag scheme the repo doesn't already use
- Don't include commit counts or file stats as a substitute for describing the change
- Don't overwrite an existing release without asking
