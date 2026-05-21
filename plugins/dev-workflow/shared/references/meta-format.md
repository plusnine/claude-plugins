# meta.md Format

## Location

`claude-output/{id}/meta.md` — one file per workflow `{id}`, shared by all flows (spec-review / spec-breakdown / bugfix / task-implement).

## Content

```
id: {id}
target-repository: {basename}
target-repository-path: {absolute path to repo's working tree root}
```

All three fields are mandatory.

## Field semantics

- `id` — workflow identifier (JIRA ticket ID or kebab-case name derived from the source).
- `target-repository` — lowercased basename of the target repo's working tree root. Identical to code-map `{repo-name}`.
- `target-repository-path` — absolute path to the target repo's working tree root. Used as the `git -C` argument for every git operation in every flow.

## Detection algorithm

Run when meta.md is absent. The result is written to meta.md.

1. Build the candidate list of git repositories rooted at or directly below the current working directory:
   a. If `git -C "$PWD" rev-parse --show-toplevel 2>/dev/null` succeeds, add the returned absolute path.
   b. For each direct subdirectory `D` of `$PWD`, if `git -C "$D" rev-parse --show-toplevel 2>/dev/null` succeeds, add the returned absolute path.
   c. Deduplicate by absolute path.
2. **One candidate**: write meta.md using that path as `target-repository-path` and its lowercased basename as `target-repository`. Do not prompt the user (no ambiguity).
3. **Two or more candidates**: present the sorted list (by absolute path) to the user and require an explicit selection. Write meta.md with the chosen entry. **Never** infer the choice from ticket prefix, spec body keywords, file names, or any other heuristic.
4. **Zero candidates**: exit immediately with an error. The error message MUST be in the language configured in `~/.claude/settings.json` (`language` key; fall back to English). English template: "No git repository was found at the current directory or any direct subdirectory. Initialize one (`git init`) or `cd` into an existing repository, then re-run the command."

## Reading

When meta.md exists, every flow MUST treat `target-repository-path` as the effective working directory for all repository-bound CLI invocations (git, gh, glab, Bitbucket REST, etc.). Implementation may use `cd "{target-repository-path}"`, `git -C "{target-repository-path}"`, or equivalent. Commands MUST NOT recompute `target-repository-path` from `$PWD`, ticket prefix, spec text, or any other heuristic.

## Validation

On read, verify all three fields are present and non-empty, and that `git -C "{target-repository-path}" rev-parse --show-toplevel` succeeds and equals the recorded `target-repository-path`. On validation failure, exit with an error in the user's configured language; do not silently re-run detection. The user must intervene (typical causes: the repo was moved, the meta.md was hand-edited incorrectly, or `claude-output/` was copied to a different workspace).

## Backward compatibility

There is no automated migration. If a checkpoint exists under `claude-output/{id}/` without `meta.md`, the next command invocation runs the detection algorithm and writes meta.md before proceeding.
