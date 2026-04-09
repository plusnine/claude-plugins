# investigation-report.md Format

## Structure

```markdown
# Investigation Report — {ticket-id}

Date: {YYYY-MM-DD}
Status: DRAFT | FINAL

## Summary
{One-paragraph summary of the bug: symptom, expected behavior, actual behavior}

## Related Feature
- Related ticket ID: {related-ticket-id} (or: none)
- Source: user-specified | none
- Spec path: claude-output/{related-ticket-id}/spec-review/source.md (or: N/A)

## Root Cause
{Technical root cause analysis: which component, what mechanism, why it fails}

## What to fix
{Business-level description of the fix — this section is the decomposition target for spec-breakdown.
Describe WHAT should change in terms of behavior, not HOW to change the code.}

## Spec Conflicts
{Summary of resolved conflicts, if any. Reference spec-conflicts.md for full resolution log.
If no conflicts: "none"}

## Affected Area
known | unknown

## Proposed Entry Points (only when area is unknown)

| File | Role |
|------|------|
```

## Status Values

- `DRAFT` — Investigation complete, conflict resolution pending or in progress
- `FINAL` — All conflicts resolved, ready for spec-breakdown consumption

Missing Status field is treated as `DRAFT`.
