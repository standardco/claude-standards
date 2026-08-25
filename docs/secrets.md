# Secrets management

## Source of truth

All secrets live in exactly two places:

- **AWS Secrets Manager** — production and staging credentials, database passwords, service API keys
- **1Password** — developer credentials, personal API tokens, anything a human needs to retrieve interactively

If a secret isn't in one of these two places, it needs to be moved there before it ships.

## What is a secret

Treat these as secrets:

- Database passwords and connection strings
- API keys and tokens (internal or external)
- OAuth client secrets
- JWT signing keys
- Encryption keys
- Webhook signing secrets
- Any value that grants access to a system or data

## What is NOT a secret

These are fine in `.env`, config files, or code:

- Database hostnames and ports (not passwords)
- Feature flags
- Log levels
- Public API base URLs
- Non-sensitive environment identifiers (`APP_ENV=staging`)

## Runtime injection

Secrets are injected at runtime — never stored on disk or in the repo.

**Local development**
```bash
# Fetch secrets from 1Password at shell startup or via wrapper script
op run --env-file=.env.1password -- <your command>
```

**CI/CD and production**
Secrets are pulled from AWS Secrets Manager via AWS SSM Parameter Store or Secrets Manager SDK calls in the application bootstrap. The IAM role attached to the container grants access — no static credentials needed.

## .env files

`.env` is for local non-secret config — ports, feature flags, log levels. It is **gitignored anyway.**

- `.env` → gitignored, local values
- `.env.example` → committed if the project keeps one: keys with empty or dummy values, no real credentials
- `.env.local` → gitignored, anything you don't want shared
- `.env.1password` → gitignored, references to 1Password items (not the secrets themselves)

Ignoring a file we permit to be non-secret looks redundant, and isn't. The permission is what guarantees the file exists in the project, and a file that exists is a file someone eventually adds one connection string to. The ignore costs nothing; the alternative depends on that never happening.

Never put a real credential in `.env`. If it's a credential, it goes in 1Password and gets injected at runtime.

## Layered controls

No single mechanism protects a credential. These are four layers with different failure modes, in the order they matter:

1. **The credential is never in the working tree.** Runtime injection via `op run` or AWS Secrets Manager. This is the actual control — everything below is damage limitation for when it isn't followed.
2. **`.gitleaks.toml` path rules block the commit.** Credential-bearing paths are declared, and committing one with values in it fails the pre-commit hook. This is the enforcement boundary.
3. **`.gitignore` reduces the accident rate.** It is not a security control. It has no effect on a file git already tracks, `git add -f` overrides it silently, its effective state depends on un-versioned per-machine config, and the credential is still sitting on disk either way.
4. **Rotation.** The only thing that works once a value is exposed.

**Why layer 2 is keyed on paths rather than on content.** gitleaks' default rules match credentials with a recognisable shape — `ghp_…`, AWS key IDs — and they work well. They do not match opaque high-entropy values: a storage-account key, a `secret_key_base`, a database passphrase. Those pass clean in any file type, a plain `.env` included. Recognising the secret is not a solvable problem. Recognising that a file we declared credential-bearing is being committed with values in it is trivial, so that is what the rules do.

A credential can be correctly gitignored, untracked, absent from history — and still sit in plaintext on disk in violation of layer 1. Layer 3 working as designed is not evidence that anything is safe.

## Pre-commit scanning

Every repo must have a secret-scanning pre-commit hook configured at `git init` time. We use [gitleaks](https://github.com/gitleaks/gitleaks).

```bash
# Install once
brew install gitleaks

# Add to .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.0
    hooks:
      - id: gitleaks
```

Pair it with a `.gitleaks.toml` at the repo root declaring the project's credential-bearing paths — that's layer 2 above, and it's what catches the values the default rules can't see. `/adopt-standards` writes one. gitleaks discovers it automatically from the repo root, so the hook needs no extra configuration.

If the hook fires, do **not** use `--no-verify` to bypass it. Fix the leak, rotate the exposed credential, then commit.

## If a secret is leaked

1. Rotate the credential immediately — assume it is compromised.
2. Revoke the old value in AWS Secrets Manager or 1Password.
3. Audit access logs for the affected service.
4. Remove the secret from git history using `git filter-repo` (not `filter-branch`).
5. Force-push the cleaned history and notify the team.

## Claude-specific rules

- Never paste a secret into a Claude prompt, even for debugging.
- Never include secrets in example code, even redacted ones with comments like `# real key goes here`.
- If Claude asks for a credential to complete a task, that's a sign the task design is wrong. Restructure it.

## AWS account details

- Production account ID: `<AWS_PROD_ACCOUNT_ID>`
- Staging account ID: `<AWS_STAGING_ACCOUNT_ID>`
- Secrets Manager region: `<AWS_REGION>`
- IAM role for local dev assume-role: `<LOCAL_DEV_ROLE_ARN>`

## 1Password vault

- Vault name: `<ONEPASSWORD_VAULT_NAME>`
- Team URL: `<ONEPASSWORD_TEAM_URL>`
