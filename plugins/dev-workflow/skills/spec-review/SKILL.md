---
name: spec-review
description: Review specifications for completeness before implementation. Use when reviewing specs, requirements, or implementation plans to ensure no gaps, overlaps, or ambiguities.
version: 0.1.0
---

## Source Cache

Before fetching spec data from JIRA/Confluence, check if `claude-output/{id}/spec-review/source.md` already exists.
- If it **exists**: ask the user whether to use the cached file or fetch fresh data (overwrite).
- If it **does not exist**: proceed to fetch from JIRA/Confluence.

## When to Use

Before presenting any implementation plan, run this review and present the plan only after all checks pass.

## Output ID Resolution

Determine `{id}` for output path `claude-output/{id}/spec-review/` using the first match below:
1. JIRA ticket ID in the **arguments** (e.g. `CA-12345`)
2. JIRA ticket ID found **directly in the spec page body** — only the page retrieved as the primary spec source (e.g. links like `https://...atlassian.net/browse/CA-12345`). Do NOT follow linked pages to find an ID.
3. Short descriptive kebab-case name derived from the spec title

## Scope Constraint

The review scope is strictly limited to the spec document itself.
- Do NOT read or reference existing code to resolve issues
- Do NOT auto-resolve issues by inferring from implementation
- Surface all issues as-is; resolution is the human's responsibility

## Output Format

`review.md` の構造・テンプレート・優先度定義・Resolution Log のルールは `references/review-format.md` を参照。

## Checklist

### 1. Source Verification
- If a primary source (docs/, design doc, etc.) is referenced, read it and verify ALL spec claims against it
- Surface discrepancies before proceeding

### 2. Coverage
- Before flagging information as missing, verify it is not already covered in another section of the same document
- All user operation paths covered (happy / error / cancel)
- All required processing per path (logging, state update, UI update)
- All spec parameters and requirements included
- Dependencies and preconditions identified (external APIs, other features, required state)
- Acceptance criteria defined (how to verify correctness)

### 3. Quality
- No gaps, overlaps, or ambiguities
- Boundary conditions and edge cases considered
- Out-of-scope items explicitly stated to prevent implicit expectations
