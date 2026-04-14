# Code Map Format

## Purpose

Index of `concept → code navigation starting points`, accumulated across task-implement and bugfix investigations.

The index answers: **"Given a goal/concept, where should exploration start to minimize wasted code scanning?"**

It is NOT a comprehensive method-map or symbol table — it records only **narrow-door entry points** (1-3 files per concept) from which an agent can navigate to the full relevant region via imports, references, and call sites.

## Location

`claude-output/_index/code-map.md` (project-scoped, gitignored with `claude-output/`).

`_index/` is a meta directory at the root of `claude-output/`, distinct from `{id}/` workflow state directories. The `_` prefix signals non-workflow scope.

## Format

TSV with a minimal header. One record per line.

```
# code-map v1
# cols: concept<TAB>starting_points<TAB>verified_at
# starting_points: pipe-separated file paths, ordered by entry priority (most useful first)
# verified_at: git short sha (7 char) at which this entry was last verified
dark mode toggle	ui/theme/Toggle.kt	a1b2c3d
auth middleware	server/middleware/Auth.kt	a1b2c3d
theme persistence	ui/theme/ThemeStore.kt|data/SettingsRepo.kt	a1b2c3d
```

### Column specification

| Column | Meaning | Constraints |
|--------|---------|-------------|
| `concept` | Lowercase natural-language phrase identifying the goal/purpose | No tab, no newline. Derived from task title (spec-breakdown flow) or ticket title (bugfix flow), normalized |
| `starting_points` | 1-3 file paths in priority order, pipe-separated | No tab, no newline. `\|` as list separator. Each path relative to repo root. Upper bound 3 — more suggests concept granularity is too coarse |
| `verified_at` | Short git sha (7-40 hex chars) at which this entry was last verified against the codebase | Must match a reachable commit |

### Reader contract

1. Lines starting with `#` are headers/comments; skip
2. Empty lines; skip
3. Other lines: split on TAB into exactly 3 fields; skip malformed rows (log warning)
4. Additional columns (future versions) are ignored by v1 readers (forward-compatible)

## Write policy

Writes are **gated by user approval**. Only validated discoveries persist.

Write triggers:
- task-implement: after Step 3 gap resolution completes (user has accepted the Affected Files)
- bugfix: after Step 2c finalization (investigation report Status: FINAL, user approval ②)

The writing orchestrator is the command, not the agent. Agents output Starting Points; commands append them to the index.

### Write operation

Append format:
```
{concept}<TAB>{file1}|{file2}<TAB>{git_short_sha}<LF>
```

- `concept`: normalized task/ticket title (lowercase, kebab/snake → space-separated, trimmed)
- `starting_points`: agent's reported Starting Points, joined with `|` in the order agent provided
- `git_short_sha`: `git rev-parse --short HEAD` at write time

Does not merge with existing entries. Same concept recorded twice creates two rows; dedup is handled at read time (most-recent verified_at wins).

Create `claude-output/_index/` if it does not exist.

## Read policy

Reads occur before the investigate agent runs (task-implement Step 2 / bugfix Step 2).

Command-side flow:

1. Load `code-map.md` (if it exists). If not, skip the read path entirely — no hints provided.
2. Extract keywords from the task prompt (task title tokens + prominent noun phrases from "What to implement" or ticket summary)
3. Filter entries: concept column substring-matches any keyword (case-insensitive)
4. For each candidate entry, verify:
   - Every path in `starting_points` exists (O(1) existence check)
   - `git diff {verified_at}..HEAD -- {starting_points}` has no material changes (if unchanged → high confidence; if changed → lower confidence but still usable as hint)
5. Entries where any path is missing → **remove from file immediately** (garbage collection on read)
6. Entries where paths exist but diff shows changes → pass to agent as lower-confidence hints (agent verifies harder)
7. Entries where paths exist and no diff → pass to agent as high-confidence hints

Duplicate entries with same concept: keep the one with the most recent verified_at (by commit recency); remove older duplicates.

## Agent integration

The investigate agent receives verified entries as `### Index Hints` in markdown table form (not raw TSV — formatted for LLM consumption):

```
### Index Hints (verified, from code-map)
| Concept | Starting Points | Confidence |
|---------|-----------------|------------|
| dark mode toggle | ui/theme/Toggle.kt | high (no diff since verified_at) |
```

The agent:
- Treats hints as **candidate starting points**, not truth
- Verifies relevance by reading
- Falls back to standard exploration if hints mislead
- Outputs a new `### Starting Points` section (1-3 files the agent deems narrow-door entries for this investigation)

The command appends the agent's Starting Points to code-map after user approval (see Write policy above).

## Invalidation (summary)

Entries become stale when:
- A listed path no longer exists → removed on next read (GC)
- `git diff {verified_at}..HEAD -- {starting_points}` shows material change → entry is kept but flagged lower-confidence; agent's own verification determines usefulness

No explicit invalidation pass is required. GC happens naturally on read.

## Versioning

Header line `# code-map v1` declares format version.

Future version bumps append columns; v1 readers silently ignore extras (forward-compatible within append-only evolution). Schema rewrites that break this contract require a major version.

## Scope limits (MVP)

- Single file per project (no module split until size becomes an issue — revisit at 1000+ entries)
- No content-hash column (git commit + path existence is sufficient)
- No `recorded_at` date column (verified_at commit provides time reference via git history)
- No cross-concept linking
- No Team-shared index (individual `.claude-output/_index/` only; explicit promotion to CLAUDE.md is a human-driven operation, not a plugin feature)
