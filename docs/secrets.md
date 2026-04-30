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

`.env` is for local non-secret config only. It should be safe to commit in a public repo.

- `.env` → committed, non-secret values
- `.env.local` → gitignored, anything you don't want shared
- `.env.1password` → gitignored, references to 1Password items (not the secrets themselves)

Never put a real credential in `.env`. If it's a credential, it goes in 1Password and gets injected at runtime.

## Pre-commit scanning

Every repo must have a secret-scanning pre-commit hook configured at `git init` time. We use [gitleaks](https://github.com/gitleaks/gitleaks).

```bash
# Install once
brew install gitleaks

# Add to .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

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
