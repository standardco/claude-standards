---
name: 1password
description: Use credentials held in 1Password without the values ever entering Claude's context — find items by metadata, write op:// references, and run commands under op run
when_to_use: Use when a task needs a credential (API key, token, service secret) to run a test, start a server, or call an API. Also triggers on "get the X from 1Password", "use the secret in the vault", or setting up a project's .env.1password
argument-hint: "[what the credential is for, or the item title]"
allowed-tools: Bash(command -v op) Bash(op --version) Bash(op account list *) Bash(op account list) Bash(op whoami *) Bash(op whoami) Bash(op vault list *) Bash(op item list *) Read Grep Glob
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

Read `## Skill Configuration` → `### 1password` for the account URL, vault and reference file. If there's no entry, ask. **Never guess an account.** With work and personal accounts both signed in, a guess reaches into the wrong one, and the item names may even match.

### 2. Preflight (metadata only)

```bash
command -v op                              # installed?
op account list                            # which accounts are configured
OP_ACCOUNT=<account-url> op whoami         # is that one unlocked?
```

| Symptom | Fix |
|---|---|
| `command -v op` prints nothing | `brew install 1password-cli`, then 1Password app → Settings → Developer → **Integrate with 1Password CLI**. Biometric unlock then replaces `op signin`. The human does this, not Claude. |
| `multiple accounts found. Use the --account flag or set the OP_ACCOUNT environment variable` | Set `OP_ACCOUNT=<account-url>` on **every** `op` command. It's per project, which is why it lives in Skill Configuration. |
| `account is not signed in` | Usually the same multiple-accounts problem. Set `OP_ACCOUNT` first. If it persists, the human unlocks the app. |

Don't `export OP_ACCOUNT` and rely on it later. Claude's shell state doesn't carry between commands, so prefix each one.

### 3. Find the item without reading values

```bash
OP_ACCOUNT=<acct> op item list --vault <vault> --format json \
  | jq -r '.[] | [.title, .category] | @tsv'
```

To see an item's fields, always filter, and **never output `.value`**:

```bash
OP_ACCOUNT=<acct> op item get "<item>" --vault <vault> --format json \
  | jq '[.fields[] | {id, label, type, section: .section.label}]'
```

`op item get` isn't pre-approved, so each call goes through a permission prompt. That's deliberate: pre-approving it would also pre-approve `--reveal`.

**Never run `op item get` on a Secure Note without a `jq` filter.** A Secure Note's body is the `notesPlain` field, and it isn't concealed. Plain `op item get <note> --format json` returns the body without `--reveal`. A LOGIN or API_CREDENTIAL item's main value is a CONCEALED field. Even so, the filter above is mandatory for every category, because one forgotten note is enough.

**Field placement differs by category.** A LOGIN keeps the secret in `password`, an API_CREDENTIAL keeps it in `credential`, and a SECURE_NOTE keeps it in `notesPlain`. If you only look for CONCEALED fields, a note comes back empty and you get no error. List the labels and types, then pick.

Whether a field *has* a value can't be established from metadata. JSON output for a concealed field may hold a placeholder rather than the value, so a non-empty `.value` proves nothing. Check presence at step 6, inside `op run`.

### 4. Write references

The project's reference file (default `.env.1password`) holds one `op://` reference per line, not values:

```bash
API_TOKEN="op://<vault>/<item>/<field>"
DB_PASSWORD="op://<vault>/<item>/password"
NOTE_TOKEN="op://<vault>/<item>/notesPlain"
```

- **Quote references.** Names may contain spaces, and quoting keeps spaces working.
- **Supported characters** are letters, digits, `-`, `_`, `.` and space. A name with anything else, such as brackets, slashes or colons, has to be referenced by its **ID** instead (`op://<vault-id>/<item-id>/<field-id>`), taken from the step 3 listing. IDs also survive renames, so prefer them for anything long-lived.
- `notesPlain` only works if the note's body is **exactly** the secret, with no label line and no trailing explanation.

Confirm the file is gitignored before writing it, using `git check-ignore -v .env.1password`. Then **read the project's own `.gitignore`**, because a global gitignore on this machine can make that check pass while the project protects nothing. The standards floor's `.env.*` covers it. References aren't secrets, but a file that exists is a file someone eventually pastes a value into.

### 5. Run

```bash
OP_ACCOUNT=<acct> op run --env-file=.env.1password -- <command>
```

`op run` resolves the references and passes values only to the child process. It masks secret values in the child's stdout and stderr by default. The CLI itself describes masking as **best effort**: a value the program transforms (base64, URL-encoded, split across writes) prints in plaintext. Don't pass `--no-masking`, and don't run a command whose purpose is to print the secret.

If the child needs to expand a variable on its own command line, run it in a subshell so `op run` substitutes before expansion:

```bash
OP_ACCOUNT=<acct> op run --env-file=.env.1password -- sh -c '<command using "$API_TOKEN">'
```

**If Claude's command is denied** (network, a permission rule such as a `curl` deny, or a sandbox), don't work around it. Write the runner, then give the human the exact command to run with `!` in the prompt. The output still lands in the conversation, and still goes through `op run`'s masking.

### 6. Verify without revealing

Run every check inside `op run`. Print lengths, counts and booleans, never values:

```bash
op run --env-file=.env.1password -- sh -c 'echo "API_TOKEN length: ${#API_TOKEN}"'
op run --env-file=.env.1password -- sh -c '[ "$TOKEN_A" = "$TOKEN_B" ] && echo same || echo different'
grep -c "<pattern>" <log>       # count, not the matching lines
```

A length of 0 means the reference resolved to an empty field, which is usually the wrong field or a note with no body. Finish by confirming that no secret value appears in any log or output file the run produced. Check with a count (`grep -cF`) inside `op run`, not by reading the file.

### 7. Missing or odd items

If an item named in a handoff, a ticket or Skill Configuration isn't there, **report the exact title and vault and ask.** Don't search for similar titles, and never create or edit items. A near-match is a different credential, possibly for a different environment, and using it is worse than stopping.

## Vault convention to recommend

When you're asked how an item should be set up, recommend this. Don't restructure a vault yourself.

- **One secret per item.**
- **LOGIN (`password`) or API_CREDENTIAL (`credential`)**, never a Secure Note body. A note's body leaks on a plain lookup (step 3).
- **Environment in the title:** `<Project> <thing> - staging` / `- production`. Prod and staging tokens look identical.
- **Only letters, digits, spaces, `-`, `_` and `.` in titles**, so `op://` references can use names. Brackets are the usual offender, and so is `(staging)`.

## What not to do

- Don't use `op read` to stdout, `--reveal`, or `--no-masking`. Each one prints the value into Claude's context.
- Don't pull a value into a shell variable that Claude's own commands then use (`TOKEN=$(op read …)`). That reimplements `op run` without the masking.
- Don't write a value to a file, not even a gitignored or scratchpad one.
- Don't put a token inline in a command you'll ask the human to approve. Approving it can write it permanently into `.claude/settings.local.json`.
- Don't guess an account or a near-match item.
