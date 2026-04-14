# Code Map Format

## Purpose

Index of `concept → code navigation starting points`, accumulated across task-implement and bugfix investigations.

The index answers: **"Given a goal/concept, where should exploration start to minimize wasted code scanning?"**

It is NOT a comprehensive method-map or symbol table — it records only **narrow-door entry points** (1-3 files per concept) from which an agent can navigate to the full relevant region via imports, references, and call sites.

## Location

`claude-output/_index/{repo-name}/code-map.md` (repo-scoped within `claude-output/`, gitignored with `claude-output/`).

`_index/` is a meta directory at the root of `claude-output/`, distinct from `{id}/` workflow state directories. The `_` prefix signals non-workflow scope.

### `{repo-name}` resolution

`claude-output/` may live at workspace level (parent of multiple repos) or inside a single repo. `{repo-name}` is resolved per-invocation so both layouts are supported.

- `basename $(git rev-parse --show-toplevel)` lowercased

No further normalization — `My_Repo-2` becomes `my_repo-2`. Filesystem-safe, information-preserving.

**Git required**: code-map operations require a git repository (commits are the verification oracle). If `git rev-parse --show-toplevel` fails, skip all code-map read and write operations entirely — the command proceeds without hints and does not persist entries. The plugin as a whole also requires git (branch/commit/PR), so this is consistent with overall assumptions.

Collision note: two distinct repos with identical basename (rare) would share a code-map. Not handled in MVP.

## Format

TSV with a minimal header. One record per line.

```
# code-map v1
# cols: concept<TAB>starting_points<TAB>verified_at
# starting_points: pipe-separated file paths, ordered by entry priority (most useful first)
# verified_at: git short sha (typically 7 chars, auto-extended if ambiguous) at which this entry was last verified
dark mode toggle	ui/theme/Toggle.kt	a1b2c3d
auth middleware	server/middleware/Auth.kt	a1b2c3d
theme persistence	ui/theme/ThemeStore.kt|data/SettingsRepo.kt	a1b2c3d
```

### Column specification

| Column | Meaning | Constraints |
|--------|---------|-------------|
| `concept` | Lowercase natural-language phrase identifying the goal/purpose | No tab, no newline. Normalization: lowercase, kebab/snake → space-separated, trimmed. Task flow: derive from `tasks/{nn}-{task-name}.md` filename with `{nn}-` prefix stripped (e.g., `01-add-dark-mode.md` → `"add dark mode"`). Bugfix flow: derive from ticket summary/title text (human-readable description, not ticket ID); additionally drop trailing period if present |
| `starting_points` | 1-3 file paths in priority order, pipe-separated | No tab, no newline. `\|` as list separator. Each path relative to repo root. Upper bound 3 — more suggests concept granularity is too coarse |
| `verified_at` | Short git sha at which this entry was last verified against the codebase | Produced by `git rev-parse --short HEAD` (7 chars minimum, auto-extended if ambiguous). Must match a reachable commit |

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

Create `claude-output/_index/{repo-name}/` if it does not exist.

### Skip condition

If the agent returned no Starting Points (empty `### Starting Points` section or "none"): skip the write entirely. An entry with no starting points provides no navigation value. This should not normally occur; if it does, the investigation itself likely failed to identify a narrow-door entry, and the command should proceed without persisting a code-map row.

## Read policy

Reads occur before the investigate agent runs (task-implement Step 2 / bugfix Step 2).

Command-side flow:

1. Load `code-map.md` (if it exists). If not, skip the read path entirely — no hints provided.
2. Extract keywords from the task prompt (task title tokens + prominent noun phrases from "What to implement" or ticket summary). Expand with semantic equivalents — common synonyms, related terms, or alternative namings likely used in the codebase for the same concept (query expansion, classical IR technique; Rocchio 1971). Use CLAUDE.md for domain vocabulary. Example: `"add dark mode persistence"` → `{dark mode, theme, night mode, appearance, persistence, settings, preferences}`
3. Filter entries: concept column substring-matches any keyword — base or expanded (case-insensitive)
4. Deduplicate: if multiple entries share the same `concept`, keep the one with the most recent `verified_at` (by git commit recency). Tiebreaker for same concept + same verified_at: keep the last-written (bottom-most) row. Remove other duplicates from the file
5. For each remaining candidate entry, verify:
   - Every path in `starting_points` exists (O(1) existence check)
   - `git diff {verified_at}..HEAD -- {starting_points}` behavior:
     - Command succeeds, no diff → high confidence
     - Command succeeds, has diff → lower confidence (still usable as hint)
     - Command fails (`verified_at` unreachable — e.g., after rebase/force-push) → treat as stale, remove entry from file
6. Entries where any path is missing → **remove from file immediately** (garbage collection on read)
7. Entries where paths exist and diff is clean → pass to agent as high-confidence hints
8. Entries where paths exist but diff shows changes → pass to agent as lower-confidence hints (agent verifies harder)

## Agent integration

The investigate agent receives surviving entries (paths exist, git commit reachable) as `### Index Hints` in markdown table form (not raw TSV — formatted for LLM consumption):

```
### Index Hints (from code-map; path existence and git reachability checked)
| Concept | Starting Points | Confidence |
|---------|-----------------|------------|
| dark mode toggle | ui/theme/Toggle.kt | high (no diff since verified_at) |
| auth middleware | server/auth/Auth.kt | lower (file changed since verified_at — verify carefully) |
```

"Confidence" reflects whether the files have changed since `verified_at`, not whether the hint is correct for the current task. High-confidence still requires the agent to verify relevance.

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
- `git diff` fails because `verified_at` is unreachable (rebase/force-push history rewrite) → removed on next read (GC)

No explicit invalidation pass is required. GC happens naturally on read.

## Versioning

Header line `# code-map v1` declares format version.

Future version bumps append columns; v1 readers silently ignore extras (forward-compatible within append-only evolution). Schema rewrites that break this contract require a major version.

## Scope limits (MVP)

- Single file per project (no module split until size becomes an issue — revisit at 1000+ entries)
- No content-hash column (git commit + path existence is sufficient)
- No `recorded_at` date column (verified_at commit provides time reference via git history)
- No cross-concept linking
- No team-shared index (individual `.claude-output/_index/` only; explicit promotion to CLAUDE.md is a human-driven operation, not a plugin feature)
- No concurrent-write coordination. Multiple sessions appending simultaneously may occasionally interleave bytes producing malformed rows; the reader contract silently skips malformed rows. File locking would be added in a later iteration if contention becomes problematic
