---
name: mcp-server
description: Audit a read-only MCP server over an existing API against the failure taxonomy that produces confidently wrong answers, and carry the build-time defaults that prevent them
when_to_use: Use when reviewing an MCP server over an existing API before or after it ships, whenever the upstream API it wraps has changed, and when starting to write one ("scaffold an MCP over this API", "wrap this API in an MCP")
argument-hint: "[optional: a tool or endpoint to scope the audit to]"
allowed-tools: Bash(git log *) Bash(git diff *) Bash(git status *) Bash(git ls-files *) Bash(git grep *) Bash(npm *) Bash(npx *) Bash(node *) Bash(ls *) Bash(find *) Bash(grep *) Bash(cat *) Read Write Edit Grep Glob Task
---

# MCP server audit

Check a **read-only MCP server over an existing API** — the shape we keep building. This is the other half of [`docs/mcps.md`](../../../docs/mcps.md), which covers when to reach for an MCP, connector versus `.mcp.json`, and how the credential is handled. That doc is about *consuming* servers. This is about the one you wrote.

The failure this exists to prevent is specific, and it is not a crash. An MCP server's consumer is a model, and a model cannot see the network. It has the tool description, the arguments it sent, and the response body — nothing else. So a server that returns a plausible `200` for a request it did not actually perform doesn't produce an error anywhere; it produces a confident wrong answer in someone's analysis, attributed to the system of record. **Every finding below is of that shape.** That is what makes them hard to catch by reading the diff.

Run this before anyone else uses the server, and again **whenever the upstream API changes** — most of the taxonomy describes a server that was correct when written and quietly stopped being correct when something upstream moved. No line of our code has to change for any of it to fire.

> This skill is generic, per the convention in [`.claude/skills/README.md`](../README.md). Nothing here names an API, a tool, or a repo. That belongs in the consuming project's own `CLAUDE.md`.

## Project context

Read `## Skill Configuration` in the project's `CLAUDE.md` first. Ask only for what isn't there:

- **Upstream API base URL, and where its credential lives** — 1Password item or SSM path, never the value
- **Endpoints in scope**
- **Whether this server is permitted to be anything other than read-only** — default **no**
- **Whether it can be run against the live API during the audit**, and with which credential — items 1 and 8 are not really answerable without it

## How to run it

Work endpoint by endpoint, not file by file. The unit of failure is a tool-plus-endpoint pair, and a file-by-file read makes the per-endpoint questions easy to skip for the endpoint that needed them.

On a server with more than three or four tools, run the sweep as parallel `bug-hunter` passes, one tool per pass, then consolidate — same pattern as [`/security-audit`](../security-audit/SKILL.md). The taxonomy goes in the prompt for each pass.

---

## 1. Filters that are accepted and not applied

**The highest-value check.** Every serious bug across our builds so far has been this one: a filter the tool accepted, did not apply, and reported as though it had.

Three observed forms, none of which errors:

- A column mapping naming a column that **does not exist upstream**. The filter is dropped, the full table comes back, and the response metadata still states the filter.
- Unrecognised query parameters **discarded silently** by the upstream API. A bogus filter value returns every row in the table, narrated as filtered.
- A filter column that is **null on every row**. The filter matches nothing, and the result still claims to be complete.

For each filter parameter on each tool, answer: does a wrong value produce an error, an empty result, or the unfiltered set? If the answer is the third, that is a finding regardless of how the code reads. Test it against the live API, not against fixtures — fixtures are the thing that hid it (see 8).

## 2. Emptiness with more than one cause

For every endpoint: **can this return zero rows for more than one reason?** The common case is an entitlement filter applied server-side that answers `200 []` rather than `403`, so "you may not see this" and "this did not happen" are the same bytes. A model reads the second one and writes it down as fact.

Every yes needs a hedge in the response body itself — not in the tool description, which the model has already stopped re-reading by the time the result arrives.

## 3. Silent truncation and partial scans

Any capped, paged, or locally-filtered result carries an explicit flag **and a sentence saying what to do about it**.

Distinguish the two claims: *this is all I looked at* and *this is all there is*. Only the second supports a total. A server that returns the first while sounding like the second is how "how many X" gets answered from a sample.

A cap added after the fact is worth a second look during the audit: by the time a result was big enough to need one, something had already read a truncated set as complete.

## 4. Errors that do not name the fix

Check that each upstream status is translated into something the model can act on, and specifically that **each message is true for every case that reaches it**.

The failure worth naming: upstream began returning `400 "does not support filtering by <param>"`, and the generic `400` handler appended *"supply `<param>`"* to every `400` — instructing the model to retry with the parameter that had just been rejected. That is an infinite-retry generator, and it was correct advice for every other `400` the handler saw.

Where a failure is **not fixable by changing arguments** — a missing or unentitled server credential, an upstream outage — the message must say so in those words. Otherwise the model treats it as an argument problem and permutes.

## 5. Hand-ported metadata

Any registry, enum, endpoint list, or column map transcribed into a constant is a finding on sight. Ours was missing 19 of 89 rows and an entire aggregation level, and nothing errored — the missing rows simply looked like data that did not exist.

Fetch it live, cache it, and let it **guide rather than gate**. A live registry that is temporarily unreachable should widen what the server accepts, not narrow it.

The exception is a value with no upstream endpoint to fetch it from — an enum that exists only in the upstream source. Duplicating it is then the only option; say so at the definition site, name where it came from, and treat the possibility of it having moved as a live risk, not a solved one.

## 6. Guardrails on stale metadata

If the server **refuses** anything on the basis of cached upstream metadata, it must re-read before refusing. A cache that only costs latency when stale is fine; a cache that produces rejections when stale converts an upstream addition into an outage lasting exactly one TTL, with a wrong error message on the way out.

## 7. Read-only asserted structurally rather than behaviourally

A test asserting no method named `post` exists on the client is easy to satisfy and easy to fool — it says nothing about a dependency that opens its own connection, or a tool added next month.

The assertion that holds:

- Record at the **fetch boundary** — the single place all outbound traffic passes through. This only works if there *is* one; a tool that makes its own call out of band is a finding in itself.
- Drive **every registered tool** through a real MCP session over an in-memory transport, via `tools/call`, not by calling internals. Enumerate the tools from `tools/list` rather than listing them in the test, so a new tool is covered on the day it is registered.
- Assert nothing but `GET` leaves, and that no `GET` carries a body.
- Assert the tool list is **non-empty**, or a registration break silently reduces the test to covering nothing.

Then verify the test **fails when a write is deliberately injected**. An assertion that has never been seen to fail is not evidence. Check `readOnlyHint: true` on every tool in the same pass — advisory rather than enforcing, but a reviewer reading `tools/list` should see the intent.

## 8. Fixtures that encode the author's assumptions

At least one fixture per endpoint captured from a **live** response. Where the code and the fixtures are written from the same belief about the upstream shape, the tests pass and the tool matches nothing in production. We shipped exactly that: a filter compared against a value the API had never once sent.

Pay particular attention to any field whose wire form was inferred from upstream source rather than observed — an enum serialised as a label where the column is an integer is the canonical version of this.

## 9. Tool descriptions that have drifted

The tool description is the **only** documentation the model gets, and the model cannot check it against anything. A stale claim there is worse than no claim: it is read as current and acted on. Check every description, parameter doc, and response note against what the code now does, and treat this as part of every change to a tool rather than as a docs task.

---

## Reporting

Rank by what a wrong answer would cost, not by how hard the fix is. For each finding: the tool and endpoint, which item above, **the concrete wrong answer it produces** — the question a user would ask and the false thing they would be told — and the fix.

Say plainly which checks were run against the live API and which only against fixtures. Items 1 and 8 are not really answerable from fixtures alone, and reporting them as clean on a fixtures-only pass restates the bug as a result.

## Writing a new one

Not a separate procedure — the audit *is* the specification, and most of what a new server needs is the items above applied forwards. What isn't already covered there:

- **Start on stdio, even if it will end up hosted.** Get tool design right before transport and auth. Tool design is what you will get wrong; transport is a known quantity you can add later.
- **Fail fast on a missing or malformed credential, at startup rather than at first call.** A server that starts and then fails every tool call looks to the model like an API problem, and it will report it as one.
- **Write a README section of real questions with the answers you actually got.** It is a regression suite, an acceptance test, and demo material at once — and writing it is usually where you discover item 1.
- **Order:** read the upstream API and capture a live response per endpoint as you go (those are the fixtures, item 8) → credential handling and the outbound client → one tool end to end, with its cap and truncation flag → the read-only test, broken on purpose to prove it fails → the remaining tools → this audit.

**One outbound client, GET only, is the load-bearing decision.** It is what makes item 7's fetch-boundary recording total rather than partial, and it is the first thing to erode when a tool needs one call that doesn't fit. Where the server is its own repo, that means a single client module and one file per tool. Where it lives inside the project whose API it wraps, follow the host project's conventions for everything else — but keep that boundary, and keep the tools out of the host's request path.

Credentials follow [`docs/secrets.md`](../../../docs/secrets.md) — injected at runtime, referenced as `${VAR}`, never written into `.mcp.json` or committed. Registering the finished server with a project follows [`docs/mcps.md`](../../../docs/mcps.md).

## Key guidelines

- **The consumer is a model with no way to check you.** It cannot see the network, cannot re-read the docs, and will not ask. Anything the response doesn't say, it will fill in confidently.
- **A `200` is not evidence the request was performed.** That sentence is most of this skill.
- **Test filters against the live API.** The fixtures were written by whoever wrote the bug.
- **Hedge in the response body, not the tool description.** By the time a result is being read, the description is long out of context.
- **Fetch metadata; don't transcribe it.** And where you must transcribe it, mark it and name the source.
- **Re-audit when upstream changes**, not when our code changes.

## What not to do

- **Don't add a write capability because a tool would be more useful with one.** Read-only is a structural property here, defended by a test; adding a write means re-deciding that, not editing a file. Raise it.
- **Don't put SDK code, versions, package names, or a file tree in this skill.** Those date, and a stale sample in a standards repo gets copied rather than questioned. Invariants and questions only — read the current SDK docs for API surface, and link a project's own server from its `## Skill Configuration` if you want a worked example.
- **Don't name a specific API, tool, endpoint, or repo here.** Project-specific context lives in the consuming project's `CLAUDE.md`. A catalog of our servers was removed from this repo once already, for reasons that still hold.
- **Don't report an audit as clean on a fixtures-only pass** without saying that is what it was.
- **Don't treat a passing read-only test as the guarantee until you've watched it fail.**
