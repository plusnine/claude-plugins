---
name: bugfix
description: Investigate codebase for root cause analysis of a bug, resolve spec-vs-implementation conflicts, and hand off to spec-breakdown and task-implement for fix implementation. Use when starting from a BTS/ITS bug ticket URL.
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
- `spec-conflicts.md` — Spec-vs-implementation conflicts and resolution log (only when conflicts exist). Reuses the same priority levels and Resolution Log structure defined in `../../shared/references/spec-gaps-format.md` (no separate format file; refer to that file for field definitions).
- `done.md` — Bugfix flow completed (post-processing done). Completion means the flow ran to the end; declined proposals do not block completion.

Downstream output (produced by orchestrated skills):
- `claude-output/{ticket-id}/spec-breakdown/` — Task decomposition
- `claude-output/{ticket-id}/task-implement/` — Implementation, commit, push, PR

## Completion States

- `done.md` — All phases complete (investigation + implementation + post-processing)
- Flow can be interrupted and resumed at any phase via checkpoint logic in `command.md`

## Spec Conflict Priority

Same as `task-implement` Spec Gap Priority (see `skills/task-implement/SKILL.md`).

## Code Map Integration

Bugfix investigations consume and contribute to the same project-scoped `concept → starting-point` index used by task-implement at `claude-output/_index/code-map.md`. Format, read/write policy, and invalidation rules: `../../shared/references/code-map-format.md`.

- **Before investigation** (Step 2): the command filters and verifies relevant index entries; verified entries are passed to `bugfix:investigate` agent as Index Hints
- **After Approval ②** (Step 2c): the command appends the agent's Starting Points as a new entry
- The index is a navigation accelerator, never source of truth — entries are always verified against code before use

## Gotchas

- Code-map operations are git-repo-gated. When `git rev-parse --show-toplevel` fails, skip all read/write steps silently.
- When the user does not provide a related-ticket-id, or the related ticket has no `spec-review/source.md`, proceed **without** a spec reference — do not block.
- 🔴 Required conflicts block transitioning `investigation-report.md` to Status `FINAL`. 🟡/⚪ conflicts do not.
- `done.md` records flow completion even if some task proposals were declined by the user during task-implement — decline does not block completion.
