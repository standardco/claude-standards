# Design baseline

We're a development-led team that still ships a lot of UI. Left without guidance, Claude produces the same generic look across models and projects, and nobody on a dev-led team is necessarily watching for it. The baseline in [`CLAUDE.md`](../CLAUDE.md#design-baseline) is a floor for that work. It doesn't set a visual direction.

## Why it lives in CLAUDE.md and not a skill

Most of our UI work isn't requested as design. Someone asks for "a settings page" or "an export screen", and nothing in the request would make Claude load a design skill. The base `CLAUDE.md` is imported into every adopted project and loaded in every session, so it's the one place a rule reliably applies to incidental UI work. Skills are copied into projects, go stale, and load only when Claude judges them relevant.

The cost is about 15 lines in every session, including backend-only projects. Keep the section short for that reason. Rationale goes here, not there.

## What it's tuned for

Our apps are mostly data-heavy: dashboards, tables, forms, admin screens and reports over public health and education data. The rules lean that way. States, tables, contrast and error copy matter more to us than hero sections. Marketing-page craft such as display type, art direction and scroll effects is deliberately left out.

## A project's design system wins

The override rule at the top of the section is the most important one. Where a project has a component library, a token set, or a `DESIGN.md`, Claude matches it, even where it breaks a rule here. The baseline exists for the blank-page case and for gaps in a project's system. It is never grounds for restyling existing UI, and the section says so itself, because this page isn't loaded in sessions.

A project that needs a different rule overrides it in its own `CLAUDE.md`, with the reason, like any other base rule.

## When a project wants more

For a product where design quality is a goal in itself, such as a client-facing site or a redesign, look at [Impeccable](https://github.com/pbakaus/impeccable). It's a design skill with commands for critique, audit and polish, and a detector hook that flags generic-AI patterns as Claude edits. It works best adopted per project with a pinned version, not inherited by everything.

It runs third-party code inside sessions that can hold credentials, so a project adopting it must:

- **Set `IMPECCABLE_LIVE_COPY_AGENT=chat`** in the `env` block of the project's committed `.claude/settings.json`, so the setting reaches every developer rather than one shell profile. By default, when no chat session is polling, live mode's Apply starts an unattended `claude` or `codex` with permission checks and sandboxing off. That agent inherits the whole environment, including injected credentials and AWS profiles, and can read anything on disk. See [`secrets.md`](secrets.md).
- **Check the engine binary, not just the skill.** The design hook runs a compiled engine on every UI edit, and that engine can be downloaded separately from the skill. Pinning the skill version doesn't establish that the engine is pinned or checksum-verified. Confirm both before adopting.

These behaviours are taken from Impeccable's [README](https://github.com/pbakaus/impeccable#readme) as of v4.9.2 (November 2026), not from its source. Re-check them when adopting a later version.

## Source

The baseline is written in our own words, drawing on Impeccable's craft floor and anti-pattern list (Apache-2.0, © Paul Bakaus), adapted for product UI.
