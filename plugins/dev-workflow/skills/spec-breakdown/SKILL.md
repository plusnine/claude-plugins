---
name: spec-breakdown
description: Decompose spec-review or bugfix investigation artifacts into coarse tasks. No code investigation — business-level decomposition only.
version: 0.2.0
model: sonnet
---

## Purpose

Transform spec-review or bugfix investigation artifacts into a coarse task list and task prompts based on business requirements alone.
Code investigation is deferred to task-implement.

## Input

One of:
- Spec-review artifacts at `claude-output/{id}/spec-review/source.md`
- Bugfix investigation report at `claude-output/{id}/bugfix/investigation-report.md` (Status: FINAL)

## Coarse Task Criteria

### Primary rule
A task is appropriately scoped if it can be described in a single sentence without "and".
- If not: split into separate tasks
- If yes: the task is valid even if it involves heavy internal processing with no user-visible outcome

### Split signals (detectable from spec text)
- Additive connectives (e.g. "also", "additionally" — or equivalent in the spec language) appear more than once → split candidate
- Sequential dependency language (e.g. "after X is done", "once X is complete" — or equivalent in the spec language) → split into before/after tasks
- Multiple distinct actors appear with independent goals (e.g., user + admin each triggering separate flows) → split candidate; actors cooperating in a single flow do not qualify

### Content rules
- Do NOT reference specific files or class names — those are determined by task-implement
- Exception: if a file/class is explicitly named in the spec, it may be referenced

## Example

Spec excerpt: *"When the user taps Save, validate the form, persist the entry, and update the list view."*

- ❌ Single task — violates the no-"and" rule: "Validate form, persist entry, and refresh list."
- ✅ Three tasks — one sentence each: (1) "Validate the save form on tap." (2) "Persist the validated entry." (3) "Refresh the list view after a successful save."

Note: actors cooperating in a single flow (user tap → form → repo → list) do NOT count as a split signal; the split here comes from three distinct operations.

## Task dependency validation

Before presenting `plan.md` for user approval, verify that the `Prerequisites` column across all tasks forms a directed acyclic graph (DAG):

1. Treat each task row in `plan.md` as a node.
2. For each `Prerequisites` cell containing `#NN`, treat it as a directed edge `NN → current task`.
3. `none` produces no edges.
4. A task listing itself (e.g., task 02 with `Prerequisites: #02`) counts as a cycle of length 1.
5. If any cycle exists, do not present `plan.md` for approval. Surface the full cycle path to the user (format: `Cycle detected: 01 → 02 → 03 → 01`), require the user to fix `plan.md`, and re-validate before re-presenting.

## Gotchas

- When source-type is `spec-review` and `review.md` Verdict is `FAIL`, do not proceed — stop and prompt the user to resolve 🔴 Required items first.
- `bugfix` flow skips the `review.md` check entirely — do not look for it under `claude-output/{id}/spec-review/`.
- File and class names belong to task-implement's investigation phase, not here. The only exception is when the spec body itself names them.
