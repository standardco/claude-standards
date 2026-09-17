---
name: mcp-server
description: Build and maintain a read-only MCP server over an existing project's API — survey what the upstream can actually do, design tools around it, and audit against the failure taxonomy that produces confidently wrong answers
when_to_use: Use when writing an MCP server against a project that already has an API ("wrap this API in an MCP", "scaffold an MCP server"), when reviewing one before or after it ships, and whenever the upstream API it wraps has changed
argument-hint: "[create | audit]"
allowed-tools: Bash(git log *) Bash(git diff *) Bash(git status *) Bash(git ls-files *) Bash(git grep *) Bash(npm *) Bash(npx *) Bash(node *) Bash(ls *) Bash(find *) Bash(grep *) Bash(cat *) Bash(mkdir *) Read Write Edit Grep Glob Task
---

# MCP server

Build and maintain a **read-only MCP server over an existing project's API** — the shape we keep building. This is the other half of [`docs/mcps.md`](../../../docs/mcps.md), which covers when to reach for an MCP, connector versus `.mcp.json`, and how the credential is handled. That doc is about *consuming* servers. This is about writing and keeping one.

Two things shape everything below.

**The consumer is a model, and it cannot see the network.** It has the tool description, the arguments it sent, and the response body — nothing else. So a server that returns a plausible `200` for a request it did not actually perform doesn't produce an error anywhere; it produces a confident wrong answer in someone's analysis, attributed to the system of record. Every item in the audit taxonomy is of that shape, which is why reading the diff doesn't catch them.

**The upstream API is the binding constraint.** These servers are thin adapters — no database connection, no query logic of their own. What the upstream can filter, page, and count decides what tools are possible and how honest their results can be. Design against that table, not against the data model behind it.

> This skill is generic, per the convention in [`.claude/skills/README.md`](../README.md). Nothing here names an API, a tool, or a repo. That belongs in the consuming project's own `CLAUDE.md`.

## Modes

Take the mode from the argument, or infer it: no server yet → `create`; a server in the repo → `audit`.

- **`create`** — build a server against a project that already has an API
- **`audit`** — check an existing one, and re-check it whenever upstream changes

They share a taxonomy. `create` is that taxonomy applied forwards, in the order the decisions actually arrive; `audit` is it applied backwards to something that already exists. Run `audit` at the end of `create`, before anyone else uses the server.

## Project context

Read `## Skill Configuration` in the project's `CLAUDE.md` first. Ask only for what isn't there:

- **The upstream project** — where its source is, if not this repo
- **Upstream API base URL, and where its credential lives** — 1Password item or SSM path, never the value
- **Endpoints in scope**
- **Whether this server is permitted to be anything other than read-only** — default **no**
- **Whether it may be run against the live API**, and with which credential — steps 1 and 2 below, and audit items 1 and 8, are not really answerable without it

---

## Mode: create

### 1. Survey what the upstream API can actually do

**Do this first and write it down as a table.** It is the design document for everything after it.

Read the upstream project's own source rather than its docs — routes file, controllers, serializers or view templates, models and their enums. Docs describe intent; the routes file describes what is deployed. For each endpoint in scope, record:

| | |
|---|---|
| **Server-side filters** | Which parameters actually narrow the result — and which are accepted and ignored |
| **Pagination** | Page size, whether it's a literal or a parameter, and what signals the last page |
| **Counts** | Whether any total is returned at all |
| **Auth** | What the credential must carry to reach this endpoint |

**Every gap in that table becomes work on our side, and specific exposure.** Where upstream cannot filter, the server fetches and filters locally — which is what drags in row caps, page-walking, and completeness reporting, and is the origin of audit items 1 and 3. Where there is no total count, no tool can honestly answer "how many". Knowing this before writing a tool is the difference between designing for it and retrofitting it.

**Record the upstream change that would remove each workaround**, next to the workaround. "Local filtering here collapses into query parameters when the controller gains them" is a standing item against the upstream project, and the alternative is a server that compensates for a limitation nobody is tracking any more.

### 2. Confirm every shape on the wire, not in the source

Step 1 tells you what exists. Only a live call tells you what is actually sent.

The canonical mistake: reading the upstream source, seeing a column compared as an integer, and building the tool to match — where the serializer calls the enum getter and puts a **string label** on the wire. Nothing errors. The filter matches nothing, the tool returns zero rows, and it reports them as a real answer.

So: call each endpoint once for real, and **keep those responses as the fixtures**. Fixtures written from the same belief as the code confirm the belief rather than the behaviour — audit item 8. Where a field's wire form was inferred rather than observed, that is the first thing to check when a tool returns nothing.

### 3. Decide the tool surface

**Tools are questions, not endpoints.** One tool per endpoint is the default that feels obvious and is usually wrong: it pushes the joining and filtering onto the model, which has to do it over whatever rows you returned, with no idea what it didn't see.

- Start from the questions people currently answer by exporting a spreadsheet. Those are the tools.
- Don't expose an endpoint just because it exists. An endpoint needing a differently-entitled credential, or with a known upstream access-control bug, is a decision to make deliberately rather than a gap to fill.
- The tool description is the **only** documentation the model gets, and it cannot check it against anything. Write it as the contract, and keep it current (audit item 9).
- Parameters the upstream cannot honour should either not exist or be applied locally and reported as such. Accepting one and dropping it is audit item 1, the most expensive bug on this list.

### 4. Settle the credential and what it can see

Establish what the server's credential is **entitled** to, not just whether it authenticates. Upstream entitlement is usually invisible from the response: a filtered-by-role endpoint answers `200 []`, not `403`, so "you may not see this" and "this did not happen" arrive as the same bytes (audit item 2).

If the server authenticates with a single shared key, **it does not model the upstream role system** — every caller sees whatever that key sees. That is a legitimate choice for an internal or demo build and an unacceptable one for a multi-user deployment, and it is invisible from the tool surface either way. Write it in the README as a limitation, in those words.

Per-user authorization is a separate decision from transport. Reaching for HTTP does not give it to you.

### 5. Build, in this order

Thin adapter: no database connection, no query logic that belongs upstream.

1. **Credential handling, and the single outbound client.** GET only, with no method capable of a write. Fail fast at startup on a missing or malformed credential — a server that starts and then fails every call looks to the model like an API problem, and it will report it as one.
2. **One tool, end to end**, with its cap and completeness reporting from the first commit. Retrofitted caps are how audit item 3 happens: by the time a result is big enough to need one, something has already read a truncated set as complete.
3. **The read-only test** — see audit item 7 for what makes it real — then **break it on purpose** and watch it fail before trusting it.
4. **The remaining tools.**

**Start on stdio even if it will end up hosted.** Tool design is what you will get wrong; transport is a known quantity you can add afterwards. Keep tool behaviour below the transport boundary so both get it.

The single outbound client is the load-bearing decision, and the first thing to erode when a tool needs one call that doesn't fit. Where the server is its own repo, that means one client module and a file per tool. Where it lives inside the project whose API it wraps, follow the host project's conventions for everything else — but keep that boundary, and keep the tools out of the host's request path.

### 6. Write down what it cannot do

Two sections, both load-bearing:

- **Limitations** — what the server doesn't model, what it can't answer, which endpoints are deliberately not exposed and why. The role model from step 4 goes here.
- **Real questions with the answers you actually got**, dated, with the credential's scope named. It is a regression suite, an acceptance test, and demo material at once — and writing it is usually where you discover audit item 1.

Then run `audit` before anyone else uses the server.

---

## Mode: audit

Run this before the server ships, and again **whenever the upstream API changes**. Most of the taxonomy describes a server that was correct when written and quietly stopped being correct when something upstream moved — no line of our code has to change for any of it to fire.

Work endpoint by endpoint, not file by file. The unit of failure is a tool-plus-endpoint pair, and a file-by-file read makes the per-endpoint questions easy to skip for the endpoint that needed them. On a server with more than three or four tools, run the sweep as parallel `bug-hunter` passes, one tool per pass, then consolidate — same pattern as [`/security-audit`](../security-audit/SKILL.md). The taxonomy goes in the prompt for each pass.

### 1. Filters that are accepted and not applied

**The highest-value check.** Every serious bug across our builds so far has been this one: a filter the tool accepted, did not apply, and reported as though it had.

Three observed forms, none of which errors:

- A column mapping naming a column that **does not exist upstream**. The filter is dropped, the full table comes back, and the response metadata still states the filter.
- Unrecognised query parameters **discarded silently** by the upstream API. A bogus filter value returns every row in the table, narrated as filtered.
- A filter column that is **null on every row**. The filter matches nothing, and the result still claims to be complete.

For each filter parameter on each tool, answer: does a wrong value produce an error, an empty result, or the unfiltered set? If the answer is the third, that is a finding regardless of how the code reads. Test against the live API — fixtures are the thing that hid it (item 8).

### 2. Emptiness with more than one cause

For every endpoint: **can this return zero rows for more than one reason?** The common case is an entitlement filter applied server-side that answers `200 []` rather than `403`. A model reads that as "this never happened" and writes it down as fact.

Every yes needs a hedge in the response body itself — not in the tool description, which the model has already stopped re-reading by the time the result arrives.

### 3. Silent truncation and partial scans

Any capped, paged, or locally-filtered result carries an explicit flag **and a sentence saying what to do about it**.

Distinguish the two claims: *this is all I looked at* and *this is all there is*. Only the second supports a total. A server that returns the first while sounding like the second is how "how many X" gets answered from a sample.

### 4. Errors that do not name the fix

Check that each upstream status is translated into something the model can act on, and specifically that **each message is true for every case that reaches it**.

The failure worth naming: upstream began returning `400 "does not support filtering by <param>"`, and the generic `400` handler appended *"supply `<param>`"* to every `400` — instructing the model to retry with the parameter that had just been rejected. An infinite-retry generator, and correct advice for every other `400` the handler saw.

Where a failure is **not fixable by changing arguments** — a missing or unentitled credential, an upstream outage — the message must say so in those words. Otherwise the model treats it as an argument problem and permutes.

### 5. Hand-ported metadata

Any registry, enum, endpoint list, or column map transcribed into a constant is a finding on sight. Ours was missing 19 of 89 rows and an entire aggregation level, and nothing errored — the missing rows simply looked like data that did not exist.

Fetch it live, cache it, and let it **guide rather than gate**. A live registry that is temporarily unreachable should widen what the server accepts, not narrow it.

The exception is a value with no upstream endpoint to fetch it from — an enum that exists only in the upstream source. Duplicating it is then the only option; say so at the definition site, name where it came from, and treat the possibility of it having moved as a live risk rather than a solved one.

### 6. Guardrails on stale metadata

If the server **refuses** anything on the basis of cached upstream metadata, it must re-read before refusing. A cache that only costs latency when stale is fine; one that produces rejections when stale converts an upstream addition into an outage lasting exactly one TTL, with a wrong error message on the way out.

### 7. Read-only asserted structurally rather than behaviourally

A test asserting no method named `post` exists on the client is easy to satisfy and easy to fool — it says nothing about a dependency that opens its own connection, or a tool added next month.

The assertion that holds:

- Record at the **fetch boundary** — the single place all outbound traffic passes through. This only works if there *is* one; a tool calling out of band is a finding in itself.
- Drive **every registered tool** through a real MCP session over an in-memory transport, via `tools/call`, not by calling internals. Enumerate from `tools/list` rather than listing tools in the test, so a new tool is covered on the day it is registered.
- Assert nothing but `GET` leaves, and that no `GET` carries a body.
- Assert the tool list is **non-empty**, or a registration break silently reduces the test to covering nothing.

Then verify the test **fails when a write is deliberately injected**. An assertion never seen to fail is not evidence. Check `readOnlyHint: true` on every tool in the same pass — advisory rather than enforcing, but a reviewer reading `tools/list` should see the intent.

### 8. Fixtures that encode the author's assumptions

At least one fixture per endpoint captured from a **live** response. Where the code and the fixtures are written from the same belief about the upstream shape, the tests pass and the tool matches nothing in production.

Pay particular attention to any field whose wire form was inferred from upstream source rather than observed — see create step 2.

### 9. Tool descriptions that have drifted

The tool description is the only documentation the model gets, and the model cannot check it against anything. A stale claim there is worse than no claim: it is read as current and acted on. Check every description, parameter doc, and response note against what the code now does, and treat this as part of every change to a tool rather than as a docs task.

### Reporting

Rank by what a wrong answer would cost, not by how hard the fix is. For each finding: the tool and endpoint, which item above, **the concrete wrong answer it produces** — the question a user would ask and the false thing they would be told — and the fix.

Say plainly which checks ran against the live API and which only against fixtures. Items 1 and 8 are not really answerable from fixtures alone, and reporting them clean on a fixtures-only pass restates the bug as a result.

Re-read the workaround list from create step 1 while you're here. An upstream limitation that has since been fixed leaves us carrying local filtering nobody needs, and that code is a standing source of item 1 and item 3 findings.

---

## Key guidelines

- **The consumer is a model with no way to check you.** It cannot see the network, cannot re-read the docs, and will not ask. Anything the response doesn't say, it will fill in confidently.
- **A `200` is not evidence the request was performed.** That sentence is most of this skill.
- **Survey the upstream before designing tools.** What it can filter, page and count is the whole design space.
- **Source tells you what exists; the wire tells you what is sent.** Confirm shapes with a live call.
- **Test filters against the live API.** The fixtures were written by whoever wrote the bug.
- **Hedge in the response body, not the tool description.** By the time a result is being read, the description is long out of context.
- **Re-audit when upstream changes**, not when our code changes.

## What not to do

- **Don't add a write capability because a tool would be more useful with one.** Read-only is a structural property here, defended by a test; adding a write means re-deciding it, not editing a file. Raise it.
- **Don't put query logic in the adapter that belongs in the upstream API.** Where you must work around a limitation, record the upstream fix next to the workaround.
- **Don't put SDK code, versions, package names, or a file tree in this skill.** Those date, and a stale sample in a standards repo gets copied rather than questioned. Invariants and questions only — read the current SDK docs for API surface, and link a project's own server from its `## Skill Configuration` as a worked example.
- **Don't name a specific API, tool, endpoint, or repo here.** Project-specific context lives in the consuming project's `CLAUDE.md`. A catalog of our servers was removed from this repo once already, for reasons that still hold.
- **Don't report an audit as clean on a fixtures-only pass** without saying that is what it was.
- **Don't treat a passing read-only test as the guarantee until you've watched it fail.**
- **Don't ship without the limitations section.** What the server doesn't model is invisible from the tool surface, and someone will assume the opposite.
