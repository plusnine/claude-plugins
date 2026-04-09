---
description: Review specifications for completeness before implementation.
argument-hint: "{source}"
---

All file output must be written in English regardless of content origin.
User-facing output (messages, previews, etc.) must be in the language specified by `language` in `~/.claude/settings.json` (fallback: English).

## Invocation

`/spec-review {source}`

Where `{source}` is a URL (Confluence, Notion, etc.) or a local file path to the specification document.

## Checkpoint

On re-run, resume position is determined by existing files:

| State | Resume from |
|-------|-------------|
| `review.md` exists & Verdict: PASS | Already complete |
| `review.md` exists & Verdict: CONDITIONAL PASS or FAIL | Step 3 (re-review with Resolution Log) |
| `source.md` exists & no `review.md` | Step 3 |
| Otherwise | Step 1 |

## Process

### Step 1: Fetch and resolve output ID

Fetch the spec content from `{source}`.

Determine `{id}` for output path `claude-output/{id}/spec-review/` using the first match below:
1. JIRA ticket ID found **directly in the spec page body** — only the page retrieved as the primary spec source (e.g. links like `https://...atlassian.net/browse/PROJ-12345`). Do NOT follow linked pages to find an ID.
2. Short descriptive kebab-case name derived from the spec title.

Present the resolved `{id}` to the user for approval before use.

### Step 2: Source Cache

Check if `claude-output/{id}/spec-review/source.md` already exists.
- If it **exists**: ask the user whether to use the cached file or overwrite with the freshly fetched data.
- If it **does not exist**: write the fetched content to `source.md`.

### Step 3: Review

If `review.md` already exists:
- Read the existing Resolution Log section.
- Items marked `✅ Resolved` must not be re-flagged in the new review.
  (They remain in the Resolution Log verbatim; do not add them to Open Items or Quality issues.)

Apply the `spec-review` skill checklist and scope constraints to the spec.

### Step 4: Output

Write to `claude-output/{id}/spec-review/`:
- `review.md` — review result. Format: `references/review-format.md`
  If a prior `review.md` exists, carry over its Resolution Log verbatim into the new file.
