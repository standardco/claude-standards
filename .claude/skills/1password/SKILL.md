---
name: 1password
description: Use credentials held in 1Password without the values ever entering Claude's context — find items by metadata, write op:// references, and run commands under op run
when_to_use: Use when a task needs a credential (API key, token, service secret) to run a test, start a server, or call an API. Also triggers on "get the X from 1Password", "use the secret in the vault", or setting up a project's .env.1password
argument-hint: "[what the credential is for, or the item title]"
allowed-tools: Bash(command -v op) Bash(op --version) Bash(op account list) Bash(op account list *) Bash(op whoami *) Bash(op vault list *) Bash(op item list *) Bash(jq *) Read Grep Glob
---

# Using 1Password Credentials

The base rules already say where credentials come from: 1Password, injected at runtime with `op run` ([`docs/secrets.md`](../../../docs/secrets.md) → "Runtime injection"). This skill is the procedure. Without it, a session that needs a token works it out by trial and error, and the obvious route is `op item get … | jq` into shell variables. That works, and it pulls the value through Claude's context to get there.

This is a skill rather than an MCP server on purpose. A 1Password MCP tool would return secret values into the model's context, which is the thing being prevented. The `op` CLI can hand a value to a child process without Claude ever seeing it. A tool call can't.

> **Project-specific context** (account, vault, reference file) belongs in the project's `CLAUDE.md` under `## Skill Configuration` → `### 1password`. Check there before asking.

## The rule

**Claude handles references, never values.**

Claude may list accounts, vaults, item titles, categories, field labels and field types. It may write `op://` references and launch commands that consume secrets.

It never prints, echoes, logs, writes to disk or pastes a secret value. That includes "just the first few characters", since a prefix is enough to tell environments apart and sometimes to identify the credential. To check a secret, look at its **length**, a **count** of matches, or the **equality** of two values. Never look at the value itself.

## Steps

### 1. Configuration

Read `## Skill Configuration` → `### 1password` for the account, vault and reference file. If there's no entry, ask. **Never guess an account.** With work and personal accounts both signed in, a guess reaches into the wrong one, and the item names may even match.

### 2. Preflight (metadata only)

```bash
command -v op                       # installed?
op account list                     # which accounts are configured
op whoami --account <acct>          # is that one unlocked?
```

| Symptom | Fix |
|---|---|
| `command -v op` prints nothing | `brew install 1password-cli`, then 1Password app → Settings → Developer → **Integrate with 1Password CLI**. Biometric unlock then replaces `op signin`. The human does this, not Claude. |
| `multiple accounts found. Use the --account flag or set the OP_ACCOUNT environment variable` | Pass `--account <acct>` on **every** `op` command. It's per project, which is why it lives in Skill Configuration. |
| `account is not signed in` | Usually the same multiple-accounts problem. Pass `--account` first. If it persists, the human unlocks the app. |

Use the flag, not an `OP_ACCOUNT=` prefix. A command that starts with a variable assignment doesn't start with `op`, so it won't match this skill's pre-approvals or anyone's permission rules. Exporting it doesn't help either, because Claude's shell state doesn't carry between commands.

### 3. Find the item without reading values

Always project `op item list` through `jq`. Its raw JSON carries `additional_information`, which for a LOGIN is the username or email.

```bash
op item list --account <acct> --vault "<vault>" --format json \
  | jq -r '.[] | [.id, .title, .category] | @tsv'
op vault list --account <acct> --format json | jq -r '.[] | [.id, .name] | @tsv'
```

To see an item's fields, filter, and **never output `.value`**:

```bash
op item get "<item>" --account <acct> --vault "<vault>" --format json \
  | jq '[(.fields // [])[] | {id, label, type, section: .section.label}]'
```

**Treat every unfiltered `op item get --format json` as printing every value.** That's established for a Secure Note's body (`notesPlain`), which came back in plain text without `--reveal`. For CONCEALED fields it hasn't been tested, and is widely reported to be the same. So the `jq` filter is the control for every category, not `--reveal` and not the item type.

`op item get` isn't pre-approved, so each call goes through a permission prompt. Pre-approving it would also pre-approve `--reveal` and the unfiltered form.

**Field placement differs by category.** A PASSWORD or LOGIN item keeps the secret in `password`, an API_CREDENTIAL keeps it in `credential`, and a SECURE_NOTE keeps it in `notesPlain`. Other categories (DATABASE, SERVER and so on) have their own labels. If you only look for CONCEALED fields, a note comes back empty and you get no error. List the labels and types, then pick.

You can't tell from metadata whether a field *has* a value. Check that at step 6, inside `op run`.

### 4. Write references

The project's reference file (default `.env.1password`) holds one `op://` reference per line, not values:

```bash
API_TOKEN="op://<vault>/<item>/<field>"
DB_PASSWORD="op://<vault>/<item>/password"
SCOPED_KEY="op://<vault>/<item>/<section>/<field>"
NOTE_TOKEN="op://<vault>/<item>/notesPlain"
```

- **Quote references.** Names may contain spaces, and quoting keeps spaces working.
- **Supported characters** are letters, digits, `-`, `_`, `.` and space. A name with anything else, such as brackets, slashes or colons, has to be referenced by its **ID** instead (`op://<vault-id>/<item-id>/<field-id>`), taken from the step 3 listings. IDs also survive renames, so prefer them for anything long-lived.
- **If a label repeats across sections**, use the section form or the field ID. Otherwise the reference is ambiguous.
- `notesPlain` only works if the note's body is **exactly** the secret, with no label line and no trailing explanation.

Confirm the file is gitignored before writing it, using `git check-ignore -v .env.1password`. Then **read the project's own `.gitignore`**, because a global gitignore on this machine can make that check pass while the project protects nothing. The standards floor's `.env.*` covers it. References aren't secrets, but a file that exists is a file someone eventually pastes a value into.

### 5. Run

```bash
op run --account <acct> --env-file=.env.1password -- <command>
```

`op run` resolves the references and passes values only to the child process. It masks secret values in the child's stdout and stderr by default. The CLI itself calls masking **best effort**: a value that's transformed (base64, URL-encoded, split across writes) prints in plain text. So:

- Don't pass `--no-masking`, and don't run a command whose purpose is to print the secret.
- Don't run verbose or tracing clients under `op run`, such as `curl -v`, `--trace`, `DEBUG=*` or HTTP wire logging. They print auth headers, and Basic auth is base64, which masking misses.
- Don't build encoded forms of the secret on the command line.
- When only success matters, send the child's output to `/dev/null` and report the exit status.

If the child needs to expand a variable on its own command line, run it in a subshell so `op run` substitutes before expansion:

```bash
op run --account <acct> --env-file=.env.1password -- sh -c '<command using "$API_TOKEN">'
```

**If Claude's command is denied** (network, a permission rule such as a `curl` deny, or a sandbox), don't work around it. Write the runner, then give the human the exact command to run with `!` in the prompt. Ask them to report pass/fail, and to paste output only if the runner keeps it to status lines, because whatever they paste lands in the conversation.

**When the permission prompt appears for `op item get`, `op read` or `op run`, approve once. Don't choose "don't ask again".** That saves a prefix rule to `.claude/settings.local.json`, and the rule then auto-approves `--reveal`, `--no-masking` and anything else after the prefix.

### 6. Verify without revealing

Run every check inside `op run`. Print lengths, counts and booleans, never values:

```bash
op run --account <acct> --env-file=.env.1password -- \
  sh -c 'echo "API_TOKEN length: ${#API_TOKEN}"'
op run --account <acct> --env-file=.env.1password -- \
  sh -c '[ "$TOKEN_A" = "$TOKEN_B" ] && echo same || echo different'
op run --account <acct> --env-file=.env.1password -- \
  sh -c '[ -n "$API_TOKEN" ] && grep -cF -e "$API_TOKEN" -- <file>'
```

- **A length of 0** means the reference resolved to an empty field, which is usually the wrong field or a note with no body.
- **Finish with the last check**, run against every log or output file the run produced. It should count 0 for each. The `-e` and `--` stop a value that starts with `-` being read as grep options, and the `-n` guard stops an empty value matching every line.
- **A multi-line value**, such as a note body, is searched one line at a time, so a count can over-report. Treat any non-zero result as a leak to investigate.

### 7. Missing or odd items

If an item named in a handoff, a ticket or Skill Configuration isn't there, **report the exact title and vault and ask.** Don't search for similar titles, and never create or edit items. A near-match is a different credential, possibly for a different environment, and using it is worse than stopping.

## Vault convention to recommend

When you're asked how an item should be set up, recommend this. Don't restructure a vault yourself.

- **One secret per item.**
- **Password (`password`) or API Credential (`credential`) for a single token.** Use a Login only when there's a real username to go with it, since its username and website fields sit empty for a bare token. A Password item is one concealed field plus a notes area for rotation dates and owners. The notes area is `notesPlain`, the same plain-text field as a Secure Note body, so it holds metadata only: never a secret, a recovery code or an old value.
- **Never a Secure Note body.** A note body needs `notesPlain`, isn't concealed in the app or in `op`'s normal output, and breaks every consumer as soon as someone adds a line of explanation.
- **Environment in the title:** `<Project> <thing> - staging` / `- production`. Prod and staging tokens look identical.
- **Only letters, digits, spaces, `-`, `_` and `.` in titles**, so `op://` references can use names. Brackets are the usual offender, and so is `(staging)`.

## What not to do

The base `settings.json` denies `op read`, `--reveal` and `--no-masking`. That's a backstop for mistakes, not permission to rely on it. Pattern rules miss reordered flags, and the rest of this list isn't enforced at all.


- Don't use `op read` to stdout, `--reveal`, `--no-masking`, or an unfiltered `op item get --format json`. Each one puts the value into Claude's context.
- Don't pull a value into a shell variable that Claude's own commands then use (`TOKEN=$(op read …)`). That reimplements `op run` without the masking.
- Don't write a value to a file, not even a gitignored or scratchpad one.
- Don't put a token inline in a command the human has to approve. Approving it can write it permanently into `.claude/settings.local.json`.
- Don't guess an account or a near-match item.
