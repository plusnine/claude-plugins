---
name: bugfix
description: Investigate codebase for root cause analysis of a bug, resolve spec-vs-implementation conflicts, and hand off to spec-breakdown and task-implement for fix implementation.
version: 0.1.0
model: sonnet
---

## Purpose

Takes a ticket from a BTS/ITS, investigates the codebase to identify the root cause,
resolves spec-vs-implementation conflicts with user input, produces an investigation report
compatible with spec-breakdown, then orchestrates spec-breakdown and task-implement to
implement the fix and create a draft pull request.

## Input

- URL to a bug ticket in BTS/ITS (e.g., `https://your-org.atlassian.net/browse/PROJ-123`)
- Related ticket ID for the original feature (optional, user-specified)
- Target codebase with CLAUDE.md containing domain knowledge

## Output

Written to `claude-output/{ticket-id}/bugfix/`:

- `meta.md` — Metadata: ticket-id, related-ticket-id, base-branch, branch-prefix. Written at branch creation.
- `investigation-report.md` — Root cause analysis with what to fix. Status: DRAFT → FINAL. Format: `references/investigation-report-format.md`
- `spec-conflicts.md` — Spec-vs-implementation conflicts and resolution log (only when conflicts exist). Reuses the same priority levels and Resolution Log structure defined in `task-implement/references/spec-gaps-format.md` (no separate format file; refer to that file for field definitions).
- `done.md` — Bugfix flow completed (post-processing done). Completion means the flow ran to the end; declined proposals do not block completion.

Downstream output (produced by orchestrated skills):
- `claude-output/{ticket-id}/spec-breakdown/` — Task decomposition
- `claude-output/{ticket-id}/task-implement/` — Implementation, commit, push, PR

## Completion States

- `done.md` — All phases complete (investigation + implementation + post-processing)
- Flow can be interrupted and resumed at any phase via checkpoint logic in `command.md`

## Spec Conflict Priority

Same as task-implement Spec Gap Priority:

| Priority | Meaning | Effect |
|----------|---------|--------|
| 🔴 Required | Must be resolved before implementation | Blocks implementation |
| 🟡 Recommended | Recommended to resolve, may proceed | Does not block |
| ⚪ Optional | Optional | Does not block |
