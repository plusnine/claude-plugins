---
name: code-map-write
description: |
  Validate an investigate agent's reply for a `### Code Map Entry` block and
  append the validated record to the per-repo code-map.jsonl index. Performs
  v1-to-v2 legacy file migration, executes the 6-layer write pipeline, appends
  metrics, and returns a single-line surface message. Use from `bugfix` Step 2c
  or `task-implement` Step 3.5; the calling command keeps retry orchestration
  and is responsible for re-invoking this skill with the retry reply.
---

# code-map-write skill

End-to-end validation and append of a single code-map entry. The full
specification of every step below lives in
`../../shared/references/code-map-format.md`; this skill is the executor.

## Args contract

The invoking command supplies these fields via the `args` payload, one per
line, colon-separated, leading/trailing whitespace trimmed:

```
target-repository-path: <absolute path to working-tree root>
agent-reply-source: <path to a file containing the agent's full reply>
ref: <workflow id; e.g. ticket id or feature id>
task: <task number for task-implement flow, or "none" for bugfix flow>
```

All four keys MUST be present. `target-repository-path` MUST match the same
field in `claude-output/{ref}/meta.md` per `meta-format.md`.

## Execution

1. **Precondition**: verify `target-repository-path` is a git working tree via
   `git -C "{target-repository-path}" rev-parse --show-toplevel`.
   - On failure: surface `Index: skipped — not a git repo`, do not write
     metrics, return.
2. **Repo resolution**: `{repo-name}` = lowercased basename of
   `target-repository-path`. All outputs root at
   `claude-output/_index/{repo-name}/`.
3. **v1 legacy detection** (only when `code-map.md` exists under
   `_index/{repo-name}/`): present the rename plan
   (`code-map.md` → `code-map.v1.deprecated[.{ISO8601-UTC}].md` on collision)
   to the user via Write Safety Gate. Execute on approval; on rejection,
   leave the v1 file untouched and continue with the v2 write.
4. **Write pipeline**: execute the 6-layer pipeline in
   `../../shared/references/code-map-format.md` "Write pipeline (multi-layer
   validation)" against the agent reply at `agent-reply-source`. All
   `git ls-files`, `git rev-parse`, and `wc -l` invocations MUST run via
   `git -C "{target-repository-path}"` to scope to the correct repo. Every
   path argument MUST be passed after `--` to prevent injection.
5. **On Layer 1 absence** (no `### Code Map Entry` heading found): surface
   `Index: skipped — no entry block`, record `appended:0,reason:no-block`
   in metrics, return.
6. **On Layer 1-5 validation failure**: do NOT retry from inside this skill.
   Surface `Index: write rejected — {layer N: reason}`, record
   `appended:0,reason:validation-failed` in metrics, return. The calling
   command decides whether to re-invoke this skill with a retry reply (see
   "Retry orchestration" below).
7. **On all layers pass**: append the validated line (with trailing `\n`)
   to `code-map.jsonl` (create the file and `_index/{repo-name}/` directory
   if absent), surface `Index: appended '{concept}' → {N entries} @
   {verified_at}` (with `auto-filled: ...` / `path normalized: ...`
   annotations per Surface outcomes), record `appended:1` in metrics,
   return.

## Retry orchestration (command-side, not in this skill)

This skill is invoked **at most twice per command step** by the calling
command:

1. First invocation with the original agent reply.
2. If the skill returns `Index: write rejected — ...`, the command performs
   the fresh agent re-invocation per `../../shared/references/code-map-format.md`
   "Retry protocol", writes the retry reply to a new file, and invokes this
   skill a second time with that file as `agent-reply-source`.
3. If the second invocation also returns `Index: write rejected — ...`, the
   command surfaces the failure to the user and proceeds with the remainder
   of the command step.

The skill never re-invokes itself, never re-invokes an agent, and never
loops on validation failure. Each invocation is one-shot.

## Metrics logging

Append a single TSV line to
`claude-output/_index/{repo-name}/code-map-metrics.log` per
`../../shared/references/code-map-format.md` "Metrics Log" → "Write
triggers". Timestamp from `date -u +"%Y-%m-%dT%H:%M:%SZ"`. Always include
`ref:{ref}`. Include `task:{task}` only when the input `task` is not the
literal string `"none"`.

## Return value

A single-line surface message string per "Surface outcomes" above. The
calling command displays this verbatim and routes its next action (proceed,
retry once, surface to user) based on whether the line starts with
`Index: appended`, `Index: skipped`, or `Index: write rejected`.

## Boundaries

- This skill does NOT invoke any LLM agent. Agent invocation (including
  retries) is orchestrated entirely by the calling command.
- This skill writes only to files under `claude-output/_index/{repo-name}/`:
  - `code-map.jsonl` (append-only)
  - `code-map-metrics.log` (append-only)
  - `code-map.md` → `code-map.v1.deprecated[.{timestamp}].md` (rename, v1
    migration only, gated by user approval via Write Safety Gate)
- All other files are read-only.
- This skill assumes git is available; the surrounding command must verify
  this before invoking the skill in environments where git may be absent.

## Reference

Full specification (schema, field semantics, auto-fix details, retry suffix
content, metrics TSV format): `../../shared/references/code-map-format.md`.
