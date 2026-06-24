---
name: user-docs
description: Generate user documentation for new features by reading the codebase (views, controllers, routes) and writing a structured guide with screenshot placeholders
when_to_use: Use when the user asks to create or update user documentation, feature guides, or end-user help docs
argument-hint: "[feature name or 'all new features']"
allowed-tools: Bash(git log *) Bash(git diff *) Grep Read Glob
---

# User Documentation Generator

Generate end-user documentation by reading the actual codebase — views, controllers, routes, and models. The output is a structured markdown guide suitable for GitHub, SharePoint wiki, or Notion.

## Inputs

Ask the user for anything not provided as arguments:
- **Features to document** — specific feature names, or "all new features" to auto-detect from git
- **Output path** — defaults to `docs/user-guide.md` (or feature-specific file like `docs/user-guide-reminders.md`)
- **Audience** — who will read this (e.g. "admin users", "all end users", "API consumers")
- **Notion page URL** (optional) — if provided, also write the documentation to this Notion page

## Steps

### 1. Identify features to document

If the user specifies features, document those. Otherwise, determine new features by comparing git history:

```bash
# What's been added since the last release
git log main..staging --oneline 2>/dev/null || git log master..staging --oneline

# Or between two tags/dates
git log --oneline --since="2 weeks ago"
```

Cross-reference with any sprint tracker, CLAUDE.md, or changelog to build the feature list.

### 2. Read the codebase for each feature

For each feature, read the relevant source files to understand exactly how it works. **Do NOT guess or hallucinate UI elements** — only document what the code actually renders.

**Key files to check:**
- **Views** — what the user sees: form fields, table columns, buttons, modals, dropdowns
- **Controllers** — what actions are available, role/permission checks, filters, redirects
- **Routes** — URL paths and HTTP methods
- **Models** — validations, scopes, status enums, business logic
- **Layouts** — navigation, menus, role-gated links
- **JavaScript** — AJAX behaviors, dynamic UI updates, confirm dialogs

For each feature, extract:
- Navigation path (how to get there from the main menu)
- Role/permission requirements (who can access it — check `before_action` filters, role checks in views, policy objects)
- Available actions (buttons, links, form submissions)
- Filter/sort options
- Table columns and what they show
- Modal dialogs and their fields
- AJAX behaviors (what updates without page reload)
- Validation rules and error messages

### 3. Write the documentation

Structure each feature section with:

```markdown
## [Feature Name]

**What it does:** [One-sentence user-facing description]

**Where to find it:** [Navigation path, e.g., Admin menu → Reports → Actions Log]

**Who can access it:** [Role/permission requirements]

### How to use it

1. [Step-by-step instructions based on the actual UI elements in the views]
2. [Reference specific form fields, buttons, dropdowns by their exact labels from the code]
3. [Note any AJAX behaviors — what updates without page reload]

[SCREENSHOT: Description of what to capture — be specific about the page state, e.g., "Actions Log page with Country filter set to 'Nigeria' and 3 result rows visible"]

### [Sub-feature, if applicable]

[Additional sections for complex features with multiple parts]

### Notes

[Edge cases, limitations, tips]
```

### 4. Add reference tables

At the end of the document, include:

**Navigation Quick Reference** — table mapping feature names to URL paths and menu locations

**Role Permissions** — table showing which roles can access which features (derive from code, not assumptions)

### 5. Write to Notion (if requested)

If the user provided a Notion page URL, use the `notion-update-page` tool to write the documentation there as well. Use `replace_content` if the page is empty, or `insert_content` to append.

### 6. Key guidelines

- **Read the code, don't guess.** Every UI element, field name, button label, and column header must come from the actual view templates. If a view uses `<%= f.label :email, "Email Address" %>`, document it as "Email Address", not "Email".
- **Screenshot placeholders must be specific.** Don't write `[SCREENSHOT: the page]`. Write `[SCREENSHOT: Reminders page showing the settings bar with Auto Emails enabled, threshold set to 7, and the filter bar with Country set to "All Countries"]`. The goal is that someone can capture exactly the right screenshot without guessing.
- **Document role restrictions accurately.** Check controller filters, view conditionals, and policy objects. If a button only renders for admin users, say so explicitly.
- **Note AJAX behaviors.** If an action works without page reload (e.g., toggling a setting, inline editing), document that the UI updates in place.
- **Keep language simple.** The audience is end users, not developers. Avoid technical jargon. Say "click the Save button" not "submit the form via POST".
- **Group related features.** If a feature spans multiple pages (e.g., a date field appears on both an edit form and a list view), document it as one feature with subsections rather than repeating it in two places.
- **Include edge cases users might hit.** E.g., "If no results match your filters, a message will display: 'No records found.'"
- **Tables for reference data.** Use tables for column descriptions, status badges, filter options, and permission matrices. They're scannable and translate well to wikis.
