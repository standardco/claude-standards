---
name: handoff
description: Summarize this session's work as a briefing for a Claude Code session in a related project, then copy it to the clipboard for pasting
when_to_use: Use when work in this project needs to be communicated to a session in another project — a new or changed API contract, a bug found in a dependency, a deploy the other side must react to. Also triggers on "notes for the other session" or "clipboard summary for <project>"
argument-hint: "[target project, e.g. the repo that consumes this one]"
allowed-tools: Bash(git log *) Bash(git diff *) Bash(git status *) Bash(git branch *) Bash(git show *) Bash(git rev-parse *) Bash(gh pr *) Bash(gh run *) Bash(uname *) Bash(pbcopy) Bash(pbcopy *) Bash(pbpaste) Bash(xclip *) Bash(xsel *) Bash(clip.exe) Read Write Grep Glob
---

# Cross-Project Handoff

Two tightly coupled projects — a service and its consumer — live in separate Claude Code sessions with no shared context. When something changes on one side, the other side finds out by accident, usually via a failure. This skill closes that gap: it turns the current session's work into a briefing the *other session* can act on immediately, and puts it on the clipboard.

The reader is a Claude Code session, not a human. That changes what belongs in it. A human skims for the gist and asks follow-up questions; the other session cannot ask you anything. Every identifier it needs to write code — exact paths, parameter names and types, status codes, error strings, URLs — must be in the text or the handoff has failed.

> Not to be confused with [`/end-of-day`](../end-of-day/SKILL.md), which writes a durable note **for tomorrow's human** in *this* project. This skill writes a transient briefing **for another project's session**, now. If both apply, run them separately — they have different readers and different content.

> **Project-specific context** (paired projects, deployment URLs, clipboard command) should be defined in the project's `CLAUDE.md` under `## Skill Configuration`. Check there first before asking the user.

## Inputs

Check the project's `CLAUDE.md` for defaults, then infer what you can before asking:

- **Target project** — who is reading this. From the argument if given; otherwise from `## Skill Configuration`, or from the conversation if only one coupled project has come up. Ask only when genuinely ambiguous, and never block on it: a handoff that names the wrong reader is still mostly correct, because the technical facts don't change.
- **Direction** — is this project the **producer** (I changed something you depend on) or the **consumer** (I found a problem in something you own)? This determines the shape of the summary. Infer it; don't ask.
- **Scope** — which slice of the session to summarize. Default to the current work: what was built, changed, fixed, or discovered since the last handoff or the start of the session.

## Steps

### 1. Establish what actually happened

Do not summarize from recollection. Sessions are long, and a plan discussed early is easy to misremember as work completed later — which is exactly the error that makes a handoff dangerous, because the other session will build against a contract that doesn't exist.

```bash
git log --oneline -20                   # what landed
git diff HEAD~<n> --stat                # shape of the change
git status --short                      # what's still local
git log @{u}..HEAD --oneline            # committed but unpushed
gh pr list --state all --limit 5        # merged, open, or neither
```

Read the actual diff for anything you are about to describe as a contract — route definitions, serializers, schemas, migrations. Quote the names from the code, not from the conversation.

### 2. Filter to what the other project can act on

A handoff is not a changelog. Include a change only if the other project has to *do* something about it, or would be wrong not to know. Internal refactors, test additions, and local tooling changes are noise here even when they were the bulk of the session's work.

Keep:
- Contract changes — new or changed endpoints, parameters, response shapes, status codes, error formats, auth requirements
- Breaking changes and removals, with the exact symptom the other side will see
- Bugs found in the other project's code, with evidence
- Deploy events that change behaviour the other side already depends on
- Fixes to bugs the other side previously reported or worked around — **especially the workarounds that should now be removed**, which is the item most often forgotten

Drop: refactors with no external effect, test-only changes, formatting, dependency bumps that don't change behaviour, anything the other project cannot observe.

### 3. Shape it by direction

**Producer → consumer** (I changed something you call). Lead with the contract. Give the full call signature: method, path, every parameter with its type and whether it's required, a realistic request, the exact response body for success and for each error case, and the base URL per environment. Say what the consumer should change and what it can leave alone.

**Consumer → producer** (I found a problem in what you own). Lead with evidence, not diagnosis. Give the exact request sent, the observed response verbatim, the expected response, when it started, and whether it reproduces. Include the reproduction command if there is one. State your hypothesis explicitly *as* a hypothesis — a guess presented as a finding sends the other session hunting in the wrong file, and it has no way to tell the difference.

### 4. Scrub before it leaves the session

The clipboard is a transfer out of this session's context. Apply the same rules as any other outbound content:

- **No PII or raw customer data** in example requests or responses. Use synthetic values shaped like the real thing — same types, same formats, same edge cases.
- **No credentials, tokens, connection strings, or API keys.** Reference where they live (`1Password → <item>`, `AWS Secrets Manager → <path>`) instead of the value.
- **No internal hostnames or infrastructure endpoints** beyond what the other project already legitimately needs to call.

Pasted log output and error payloads are the usual leak: they often carry real record IDs, emails, or auth headers. Redact them rather than trimming the log — a redacted log still reproduces the bug.

### 5. Copy to the clipboard and show it

Write the summary to a file in the scratchpad directory, then pipe it to the platform clipboard command. Going via a file avoids shell-quoting damage to the markdown — backticks, quotes, and newlines survive intact.

```bash
pbcopy < <file>                              # macOS
xclip -selection clipboard < <file>          # Linux (X11) — or: xsel --clipboard --input
wl-copy < <file>                             # Linux (Wayland)
clip.exe < <file>                            # WSL
```

Detect with `uname -s` if the platform isn't already known. If no clipboard tool is available, say so plainly and display the summary — the summary is the deliverable, the clipboard is the convenience.

Then display the full summary in the response. The user reviews it before pasting, and that review is the only check on this skill's output.

## Summary structure

```markdown
# <Project>: <what changed, in a phrase> (<status>)

Handoff from the `<this-project>` session. <One sentence on why the reader
is getting this — what they need to do or know.>

## What changed
<Technical detail sufficient to act without follow-up questions. Exact
identifiers, copied from the code. For a contract: full signature, request
and response bodies, error cases. For a bug: request sent, response
observed, response expected, reproduction.>

## Status
- **Deployed:** <what is live, where, at which commit>
- **Merged, not deployed:** <what's on the main branch but not out>
- **Local only:** <what exists nowhere but this machine — the reader must not build against it>
- **Known issues:** <what is still broken, and any workaround that is still required>

## What this means for <target project>
1. <Specific action — file or component, and what to change>
2. <Workarounds that can now be removed>
3. <What deliberately does not need to change>

## Verified vs assumed
- Verified: <what was actually tested or observed, and how>
- Assumed: <what was inferred and would need checking>
```

Drop sections that don't apply. Keep it as short as it can be while still being actionable — but never drop an identifier to save a line.

## Key guidelines

- **Exact identifiers, always.** `GET /records/{id}/status?format=<str>&strict=<bool>` is actionable. "the new status endpoint" is not. The other session will type what you write.
- **Status is not a footnote.** *Deployed* / *merged* / *local only* are three completely different instructions to the reader. Naming the wrong one sends them to integrate against something that isn't there.
- **Separate verified from assumed.** The reader will treat everything in the text as fact unless told otherwise, and it has no way to check.
- **Say what not to do.** "Your retry-on-404 workaround can be removed" and "your request shape doesn't change" prevent unnecessary work as effectively as an instruction creates useful work.
- **Name files and commits.** `app/services/<module>.py:42`, commit `a1b2c3d` — the reader may have access to the same repos.
- **One handoff, one subject.** If the session produced two unrelated things the other project must act on, two summaries beat one that buries the second.

## What not to do

- Don't describe work as deployed, merged, or complete without checking — the failure mode is the other project building against something that doesn't exist yet
- Don't paste raw logs, fixtures, or payloads without scrubbing them first
- Don't write for a human — no motivation, no narrative of how the bug was found, no praise for the fix
- Don't include the parts of the session the other project cannot observe
- Don't guess at the other project's internals; describe your side precisely and let its session decide what to change
- Don't overwrite the clipboard without showing what was put there
