# Data privacy

## Why this matters

We work with public health and education data. Individuals in our datasets are often in vulnerable situations. A privacy breach is not just a compliance event — it causes real harm. These rules are non-negotiable.

## The rule

**No raw customer, patient, student, or PII data ever enters a Claude workflow.**

This includes:
- Prompts you type
- Files you attach
- Clipboard pastes
- Data you ask Claude to analyze or transform

If you are unsure whether something is PII, treat it as PII.

## What counts as PII

Direct identifiers:
- Names, initials
- Dates of birth, ages combined with other data
- Geographic data below county level (street address, ZIP code, GPS coordinates)
- Phone numbers, email addresses
- Social Security numbers, student IDs, medical record numbers
- IP addresses, device IDs, cookie identifiers
- Photos, biometrics

Quasi-identifiers (individually harmless, dangerous in combination):
- Race/ethnicity
- Diagnosis codes or condition flags
- School or clinic name combined with grade/age
- Employment status combined with location and age

When in doubt, treat it as a direct identifier.

## De-identification before Claude

Every project that handles real data must document its de-identification process in its own `CLAUDE.md`. The process must cover:

1. **What fields are removed** — list every column dropped before data touches Claude
2. **What fields are transformed** — e.g., date-of-birth → age bucket, ZIP → region
3. **What tool or script performs the transformation** — path to the script or name of the pipeline step
4. **How synthetic test data is generated** — tool, seed, and location of synthetic datasets

### Minimum safe de-identification for tabular data

```
Remove:  name, email, phone, address, zip, dob, ssn, mrn, student_id, ip_address, device_id
Bucket:  age → decade (20s, 30s, …), date → year+quarter, zip → state or region
Replace: free-text fields with synthetic equivalents before passing to Claude
```

### Minimum safe de-identification for documents

- Replace names with `[NAME]` or a consistent pseudonym within the document
- Replace dates with relative offsets (`T+0`, `T+14`, …)
- Replace addresses and phone numbers with `[REDACTED]`
- Replace IDs (SSN, MRN, student ID) with `[ID-REDACTED]`

Use a reversible pseudonymization process if you need to re-link after Claude processing. Store the mapping outside Claude.

## Synthetic and anonymized datasets for dev/test

Development and testing must use synthetic or fully anonymized data. Never use production data, even in a dev environment, even "just to check something quickly."

Recommended synthetic data tools:

- [Faker](https://faker.readthedocs.io/) — general-purpose fake data in Python/JS
- [Synthea](https://github.com/synthetichealth/synthea) — synthetic patient records
- [SDV (Synthetic Data Vault)](https://sdv.dev/) — statistically representative synthetic tabular data

Document the synthetic dataset generation scripts in the project repo.

## Claude-specific rules

- Claude does not retain data between sessions, but treat every prompt as potentially logged. Write prompts accordingly.
- Do not use real data as few-shot examples in a prompt. Use synthetic examples.
- If a task requires Claude to see a sample of real data to understand the schema, describe the schema in prose or use a synthetic row — do not paste a real row.
- Data analysis workflows should pass only aggregated or anonymized results to Claude, not record-level data.

## Incident response

If real PII enters a Claude session:

1. End the session immediately.
2. Document what was shared: session timestamp, approximate data volume, fields exposed.
3. Notify `<PRIVACY_CONTACT_EMAIL>` within 24 hours.
4. Follow the breach notification process at `<BREACH_RUNBOOK_URL>`.

## Compliance references

- `<APPLICABLE_REGULATION>` (e.g., HIPAA, FERPA, COPPA) — `<REGULATION_SUMMARY_URL>`
- Data Processing Agreement with Anthropic: `<DPA_URL>`
- Internal privacy policy: `<INTERNAL_PRIVACY_POLICY_URL>`
