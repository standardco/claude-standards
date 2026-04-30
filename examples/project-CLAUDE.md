<!--
  Template for a project-level CLAUDE.md.
  1. Copy this file to your project root as CLAUDE.md.
  2. Replace all <PLACEHOLDERS>.
  3. Remove sections that don't apply.
  4. Add project-specific rules below the import.
-->

@./claude-standards/CLAUDE.md

# <Project Name> — Project-specific rules

## What this project is

<One paragraph: what the service does, what stack it uses, who it serves.>

## Repo structure

```
<PROJECT_ROOT>/
├── src/
├── tests/
├── docker/
└── ...
```

Key files:
- `<PATH>` — <what it does>
- `<PATH>` — <what it does>

## Stack

- Language: `<LANGUAGE_AND_VERSION>`
- Framework: `<FRAMEWORK>`
- Database: `<DATABASE>`
- Test runner: `<TEST_RUNNER>`
- Package manager: `<PACKAGE_MANAGER>`

## Commands

```bash
# Bootstrap
<BOOTSTRAP_COMMAND>

# Run tests
<TEST_COMMAND>

# Run locally
<DEV_COMMAND>

# Lint
<LINT_COMMAND>
```

## De-identification

<!-- Required. Describe how real data is stripped before entering any Claude workflow. -->

**Fields removed before data touches Claude:**
- `<FIELD_NAME>` — reason
- `<FIELD_NAME>` — reason

**Fields transformed:**
- `date_of_birth` → age bucket (decade)
- `zip_code` → state/region
- `<FIELD>` → `<TRANSFORMATION>`

**De-identification script:** `<PATH_TO_SCRIPT>`

**Synthetic test data:** Generated with `<TOOL>`. Seed datasets at `<PATH>`.

## Conventions

<!-- Add project-specific rules the style-enforcer should know about. -->

- <Convention 1>
- <Convention 2>

## Prohibited patterns

<!-- Things Claude must not do in this project. -->

- Do not use `<BANNED_LIBRARY>` — we use `<PREFERRED_LIBRARY>` instead.
- Do not modify `<SENSITIVE_FILE>` without a second review.

## Known gotchas

<!-- Surprising things about this codebase that would trip someone up. -->

- <Gotcha 1>
- <Gotcha 2>
