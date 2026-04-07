---
description: Review specifications for completeness before implementation.
argument-hint: "{id}"
---

## Invocation

`/spec-review {id}`

Where `{id}` is a JIRA ticket ID (e.g. `CA-12345`) or a Confluence URL.

## Process

### Step 1: Source Cache

Before fetching spec data from JIRA/Confluence, check if `claude-output/{id}/spec-review/source.md` already exists.
- If it **exists**: ask the user whether to use the cached file or fetch fresh data (overwrite).
- If it **does not exist**: proceed to fetch from JIRA/Confluence.

### Step 2: Output ID Resolution

Determine `{id}` for output path `claude-output/{id}/spec-review/` using the first match below:
1. JIRA ticket ID in the **arguments** (e.g. `CA-12345`)
2. JIRA ticket ID found **directly in the spec page body** — only the page retrieved as the primary spec source (e.g. links like `https://...atlassian.net/browse/CA-12345`). Do NOT follow linked pages to find an ID.
3. Short descriptive kebab-case name derived from the spec title

### Step 3: Review

Apply the `spec-review` skill checklist and scope constraints to the spec.

### Step 4: Output

Write to `claude-output/{id}/spec-review/`:
- `source.md` — cached spec content
- `review.md` — review result. Format: `references/review-format.md`
