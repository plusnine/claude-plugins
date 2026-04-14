# {nn}-spec-gaps.md Format

## Structure

```
# Spec Gaps — Task {nn}: {task name}

Date: {YYYY-MM-DD}
Status: OPEN | RESOLVED
```

## Open Items

Include unresolved items only. Resolved items must be moved to the Resolution Log.

```
## Open Items

| # | Priority | Requirement | Description |
|---|----------|-------------|-------------|
| SG-1 | 🔴 Required | {requirement name} | {ambiguity, contradiction, or missing definition} |
| SG-2 | 🟡 Recommended | {requirement name} | ... |
```

Priority definitions:

| Priority | Meaning | Effect on status |
|----------|---------|------------------|
| 🔴 Required | Must be resolved before implementation | Keeps Status OPEN |
| 🟡 Recommended | Recommended to resolve, may proceed | No effect on status |
| ⚪ Optional | Optional | No effect on status |

Status is `RESOLVED` when no 🔴 Required items remain in Open Items.

## Resolution Log

Append entries as issues are resolved. Never delete or overwrite existing entries.

```
## Resolution Log

| ID | Round / Date | Response & Reason | Status |
|----|-------------|-------------------|--------|
| SG-1 | 2 / 2026-04-03 | Controlled server-side, no spec change needed — confirmed with PM | ✅ Resolved |
| SG-2 | 2 / 2026-04-03 | Under confirmation with server team | 🔄 Pending |
```

Rules:
- `✅ Resolved` — reason required; record who confirmed and what the outcome was.
- `🔄 Pending` — reason required; record who is confirming.
- `🔄 Pending` on 🔴 Required items keeps Status OPEN.
- `Round` — determined by the maximum Round value in existing Resolution Log entries + 1. First round is 1.
