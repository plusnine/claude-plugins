# Code Map Format (v2)

## Purpose

Index of `concept → code navigation starting points`, accumulated across task-implement and bugfix investigations.

The index answers: **"Given a goal/concept, where should exploration start to minimize wasted code scanning?"**

It is NOT a comprehensive method-map or symbol table — it records only **narrow-door entry points** (1-5 files per concept) from which an agent can navigate to the full relevant region via imports, references, and call sites.

## Audience

This file is **AI/LLM-optimized** and not intended for human reading or hand-editing. It is gitignored (via `claude-output/`). All read and write operations go through the plugin's command layer.

## Location

`claude-output/_index/{repo-name}/code-map.jsonl`

`_index/` is a meta directory at the root of `claude-output/`, distinct from `{id}/` workflow state directories. The `_` prefix signals non-workflow scope.

### `{repo-name}` resolution

`claude-output/` may live at workspace level (parent of multiple repos) or inside a single repo. `{repo-name}` is resolved per-invocation so both layouts are supported.

- `basename $(git rev-parse --show-toplevel)` lowercased

No further normalization — `My_Repo-2` becomes `my_repo-2`. Filesystem-safe, information-preserving.

**Git required**: code-map operations require a git repository (commits are the verification oracle). If `git rev-parse --show-toplevel` fails, skip all code-map read and write operations entirely — the command proceeds without hints and does not persist entries.

Collision note: two distinct repos with identical basename (rare) would share a code-map. Not handled in MVP.

## Format

**JSON Lines** (one JSON object per line, LF-terminated, UTF-8).

- **No header, no comments, no blank lines.** Every non-empty line MUST be a valid JSON object conforming to the schema below. Lines that fail to parse are errors, not silently skipped.
- **File terminator**: the file MUST end with a single LF (`\n`). CRLF is not allowed.
- **Line length**: each record MUST be ≤ 4096 bytes. Under Linux (ext4/XFS) and macOS (APFS), single-record appends at this size are empirically atomic via `O_APPEND`. POSIX formally guarantees atomicity only for pipe/FIFO writes of `PIPE_BUF` or less — regular-file atomicity is implementation-dependent. This limit is therefore a pragmatic ceiling for multi-session write safety, not a spec-level guarantee; truly concurrent writers are out of scope (see "Scope limits (MVP)").
- **Ordering**: records are append-only in write order; dedup and GC happen at read time.

### Canonical example (2 records)

```jsonl
{"concept":"dark mode toggle","aliases":["night mode","theme toggle","appearance mode"],"tags":["ui","theme","settings"],"entries":[{"path":"ui/theme/Toggle.kt","symbol":"ThemeToggle","kind":"class","anchor":"L12-L80","summary":"user-facing switch; delegates persistence to ThemeStore."},{"path":"ui/theme/ThemeStore.kt","symbol":"ThemeStore.setMode","kind":"function","anchor":null,"summary":"applies theme change; writes to SettingsRepo."}],"verified_at":"a1b2c3d"}
{"concept":"auth middleware","aliases":["authentication","authn"],"tags":["server","security"],"entries":[{"path":"server/middleware/Auth.kt","symbol":"AuthMiddleware","kind":"class","anchor":"L20-L150","summary":"verifies JWT and injects user context into request."}],"verified_at":"a1b2c3d"}
```

## Schema (JSON Schema Draft 2020-12)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "code-map-entry-v2",
  "type": "object",
  "required": ["concept", "aliases", "tags", "entries", "verified_at"],
  "additionalProperties": false,
  "properties": {
    "concept": {
      "type": "string",
      "pattern": "^[a-z0-9]+( [a-z0-9]+)*$",
      "minLength": 1,
      "maxLength": 80
    },
    "aliases": {
      "type": "array",
      "items": {
        "type": "string",
        "pattern": "^[a-z0-9]+( [a-z0-9]+)*$",
        "minLength": 1,
        "maxLength": 80
      },
      "maxItems": 10,
      "uniqueItems": true
    },
    "tags": {
      "type": "array",
      "items": {
        "type": "string",
        "pattern": "^[a-z0-9][a-z0-9-]*$",
        "minLength": 1,
        "maxLength": 24
      },
      "maxItems": 8,
      "uniqueItems": true
    },
    "entries": {
      "type": "array",
      "minItems": 1,
      "maxItems": 5,
      "items": {
        "type": "object",
        "required": ["path", "symbol", "kind", "anchor", "summary"],
        "additionalProperties": false,
        "properties": {
          "path": {
            "type": "string",
            "pattern": "^(?!.*\\\\)(?!.*:)[^\\s/][^\\n\\t]*$"
          },
          "symbol": {
            "oneOf": [
              {"type": "string", "pattern": "^[A-Za-z_][A-Za-z0-9_.:]*$"},
              {"type": "null"}
            ]
          },
          "kind": {
            "enum": ["class", "function", "module", "interface", "file", null]
          },
          "anchor": {
            "oneOf": [
              {"type": "string", "pattern": "^L\\d+(-L\\d+)?$"},
              {"type": "null"}
            ]
          },
          "summary": {
            "type": "string",
            "pattern": "^[^\\n]{1,179}\\.$"
          }
        }
      }
    },
    "verified_at": {
      "oneOf": [
        {"type": "string", "pattern": "^[0-9a-f]{7,40}$"},
        {"type": "null"}
      ]
    }
  }
}
```

### Key order (enforced in agent output and writer)

Top-level: `concept, aliases, tags, entries, verified_at`
Entry: `path, symbol, kind, anchor, summary`

Key omission is not allowed — optional values MUST be present as `null` (never `""` or missing).

### Field semantics

| field | required | semantics |
|---|---|---|
| `concept` | ✅ | Primary search key. Natural-language phrase (lowercase ASCII words separated by single spaces). Derived from task-prompt filename or ticket title. |
| `aliases` | ✅ (may be empty `[]`) | Synonyms / alternate phrasings. Pre-populated at write time so readers do not need LLM query expansion. |
| `tags` | ✅ (may be empty `[]`) | Free-form classification labels (domain, layer, role-like). Dual-axis usage encouraged: classification (`ui`, `server`, `payment`) + role-like (`entry`, `logic`, `state`, `io`). Pattern enforces lowercase hyphen-separated form. |
| `entries` | ✅ | 1-5 files, ordered by exploration priority (index 0 = read first). |
| `entries[].path` | ✅ | Repo-root relative POSIX path. No leading `/`, no backslash `\`, no colon `:`. Case-sensitive. |
| `entries[].symbol` | ✅ (string or null) | Class/function/method name. Dot or colon for scoping (`Foo.bar`, `mod::fn`). `null` if file-level or no single-symbol anchor. |
| `entries[].kind` | ✅ (enum or null) | Structural category. Language-neutral enum: `class \| function \| module \| interface \| file`. `null` if none applies. |
| `entries[].anchor` | ✅ (string or null) | Line range `L{N}` or `L{N}-L{M}` (1-based, start ≤ end). `null` for file-level hint. |
| `entries[].summary` | ✅ | One sentence, ≤ 179 chars + period. No newlines. Lets agents decide hit/miss before reading. |
| `verified_at` | ✅ (string or null) | Git short SHA (7-40 hex chars) at which the record was verified. Agent output uses `null`; command substitutes the actual SHA at write time. |

### Tag conventions (guidance, not enforced by schema)

Tags serve two axes simultaneously:
- **Classification axis**: domain / layer / module (e.g. `ui`, `server`, `payment`, `auth`)
- **Role-like axis** (optional): what the code does (e.g. `entry`, `logic`, `state`, `io`, `config`, `test`, `util`)

Prefer project-specific vocabulary when it clarifies intent (`viewmodel`, `composable`, `handler`, `trait`, `hook`). Pattern `^[a-z0-9][a-z0-9-]*$` is the only enforced rule; choose short tags already used in the codebase over generic synonyms.

## Agent output contract

The investigate agent MUST end its response with the following section:

~~~markdown
### Code Map Entry

```code-map
{SINGLE-LINE JSON OBJECT}
```
~~~

**Heading level note**: the raw agent reply uses `### Code Map Entry` (H3). When the bugfix command embeds this section into `investigation-report.md`, the heading is elevated to `## Code Map Entry` (H2) to match the report's heading hierarchy. The fence content and schema are identical; only the heading level differs.

Strict rules:
1. **`### Code Map Entry` is the final heading of the response.** Nothing follows the closing code fence except EOF or a trailing blank line.
2. **Exactly one fenced code block** with the language identifier `code-map`.
3. **Exactly one line** inside the fence (no embedded newlines; no leading/trailing whitespace lines).
4. **Valid JSON** parseable by `JSON.parse`: double quotes only, no trailing comma, no single quotes, no comments.
5. **`verified_at` MUST be `null`** (the command injects the git SHA at write time).
6. **All keys present** in the fixed order stated above; optional values as `null`, never omitted.
7. **ASCII-only** in `concept`, `aliases`, `tags` (schema pattern enforces).
8. **Starting points MUST be existing, git-tracked files**. Do not include files scheduled for creation or generated build artifacts.
9. **Entries priority order**: index 0 = highest priority (most likely read-first for a future investigator).

### Self-check before emitting

Walk through mentally before writing the fence:
1. Does it `JSON.parse` cleanly? (double-quote only, no trailing comma, proper escapes)
2. Are all required keys present with correct types?
3. Do all `pattern` constraints pass? (`concept`, `aliases`, `tags`, `path`, `symbol`, `anchor`, `summary`)
4. Is `concept` distinct from every entry in `aliases`?
5. Within `entries`, is every `(path, symbol)` pair unique?
6. Does every `summary` end with a period, contain no newline, and ≤ 179 chars + `.`?
7. Is every `entries[].path` currently present in `git ls-files`?
8. Does every `anchor` satisfy `start ≤ end` and `end ≤ file line count`?
9. Is `verified_at` exactly `null`?

If any check fails, rewrite before emitting.

### Positive examples

```code-map
{"concept":"dark mode toggle","aliases":["night mode","theme toggle"],"tags":["ui","theme"],"entries":[{"path":"ui/theme/Toggle.kt","symbol":"ThemeToggle","kind":"class","anchor":"L12-L80","summary":"user-facing switch delegates persistence to ThemeStore."}],"verified_at":null}
```

```code-map
{"concept":"auth middleware","aliases":[],"tags":["server","security","entry"],"entries":[{"path":"server/middleware/Auth.kt","symbol":"AuthMiddleware","kind":"class","anchor":"L20-L150","summary":"verifies JWT and injects user context into request."},{"path":"server/middleware/TokenParser.kt","symbol":"TokenParser","kind":"class","anchor":null,"summary":"parses and validates JWT signature and claims."}],"verified_at":null}
```

### Negative examples (each is rejected)

1. **Text outside fence** — any prose between the heading and the opening fence, or after the closing fence.
2. **Multiple fences** — two or more `code-map` fences in the response.
3. **Multi-line JSON** — `\n` inside the fence (pretty-printed JSON is rejected).
4. **verified_at not null** — agent output with `"verified_at":"a1b2c3d"` is rejected.
5. **Missing period in summary** — `"summary":"does a thing"` is rejected.
6. **Key omitted** — writing `"kind"` and `"anchor"` only when non-null; all optional keys must be present with `null`.
7. **Trailing comma or single quotes** — `{'concept':'x',}` is rejected.
8. **Non-ASCII in concept/aliases/tags** — `"concept":"ダークモード"` is rejected; express in ASCII or translate.

## Write policy

Writes are **gated by user approval at the surrounding command step** (task-implement Step 3.5, bugfix Step 2c). Only validated discoveries persist.

Write triggers:
- task-implement: after Step 3 gap resolution completes (user has accepted the Affected Files)
- bugfix: after Step 2c finalization (investigation report Status: FINAL, user approval ②)

The writing orchestrator is the command, not the agent. The agent emits a `### Code Map Entry` block; the command validates, injects `verified_at`, and appends.

### Write pipeline (multi-layer validation)

Every write goes through six layers. Failure at any layer triggers retry (see below).

1. **Extraction**: from the agent's response, locate exactly one `### Code Map Entry` heading whose body contains exactly one fenced block with language identifier `code-map`.
2. **Line structure**: the fence contains exactly one non-empty line, ≤ 4096 bytes, no embedded newlines or control characters other than the single terminating context.
3. **JSON syntax**: the line `JSON.parse`s without error.
4. **Schema conformance**: the parsed object validates against the JSON Schema above (type, pattern, enum, required, `additionalProperties:false`).
5. **Semantic validation**:
   - `concept` does not appear in `aliases`
   - within `entries`, `(path, symbol)` tuple is unique across entries
   - for each `entries[].path`: `git ls-files -- {path}` returns a non-empty result (tracked by git)
   - for each `entries[].anchor` (when non-null): parse `L{start}(-L{end})?` and verify `start ≥ 1 ∧ end ≥ start ∧ end ≤ (wc -l {path})`
   - every `entries[].summary` is distinct from every other entry's summary within the same record
6. **Verified-at injection**: replace `"verified_at":null` with `"verified_at":"{git rev-parse --short HEAD}"`.

On success, append the line to `code-map.jsonl` (creating the file and the `_index/{repo-name}/` directory if needed). Ensure the appended line ends with `\n`.

### Retry protocol

On failure at any of layers 1-5, the command MUST retry exactly once by re-invoking the agent with the following additional prompt appended to its original input (literal text, do not paraphrase):

```
## Previous output was rejected

Layer: {1|2|3|4|5}
Reason: {exact error message from the validator}
Offending substring: {literal excerpt, max 200 chars, first occurrence}

Produce a corrected `### Code Map Entry` block. All other requirements from the original prompt still apply.
```

If the second attempt also fails, the command MUST NOT append anything. Surface the final failure reason to the user, record `appended:0,reason:validation-failed` in the metrics log, and proceed with the remainder of the command step (downstream steps are not blocked by code-map append failure).

### Skip (not a failure)

If the agent's response contains **no** `### Code Map Entry` heading at all, treat it as an intentional skip (e.g., the investigation concluded there is no useful starting point to record). The command does not retry, surfaces `Index: skipped — no entry block` to the user, and records `appended:0,reason:no-block` in the metrics log.

The agent may choose to skip in the following cases:
- No meaningful starting point was identified (e.g., the investigation rejected the task as unnecessary).
- All candidate starting points are not yet git-tracked (e.g., the fix creates entirely new files).

Skipping is a graceful non-append; it is not an error.

## Read policy

Reads occur before the investigate agent runs (task-implement Step 2 / bugfix Step 2).

Command-side flow:

1. Resolve `{repo-name}`. If `git rev-parse --show-toplevel` fails: skip the entire read path (no hints).
2. Load `code-map.jsonl`. If not present: skip (no hints).
3. **Legacy v1 detection**: if `code-map.md` exists in the same directory, schedule a rename approval (see "v1 → v2 migration" below). This detection happens on first v2 write, not on every read — readers silently ignore `code-map.md`.
4. Parse each line as JSON. Parse-error lines are malformed and **surfaced to the user** (`Index: N malformed lines skipped — investigate code-map.jsonl`); they are not silently ignored but do not block the read.
5. Schema-validate each record. Records failing validation are also surfaced and skipped (readers are forgiving; writers are strict).
6. Extract keywords from the task prompt or ticket (task title + prominent noun phrases).
7. Match: case-insensitive substring match of any keyword against `concept`, any element of `aliases`, or any element of `tags`.
8. Deduplicate: for records sharing the same `concept`, keep the one with the most recent `verified_at` (by git commit recency). Tiebreaker: keep the last-written (bottom-most) record. Remove older duplicates from the file as garbage collection.
9. For each surviving record, verify:
   - Every `entries[].path` exists on disk (else remove the record from the file immediately as GC)
   - `git diff {verified_at}..HEAD -- {all paths in the record}`:
     - success + no diff → high confidence
     - success + diff → lower confidence (still usable)
     - failure (`verified_at` unreachable, e.g., after force-push) → treat as stale, remove record from file
10. Forward surviving records to the agent as Index Hints (see next section).

### Forward compatibility (Postel's law)

Readers MUST silently ignore unknown top-level keys or unknown `entries[]` keys added by future schema versions, as long as all currently-required keys are present and validate. Writers remain strict: they MUST NOT emit keys outside the current schema.

This allows additive schema evolution without breaking old readers.

## Agent integration (Index Hints delivered to agent)

The investigate agent receives surviving records as a markdown table (not raw JSONL — formatted for LLM consumption):

```markdown
### Index Hints (code-map v2; path existence and git reachability checked)

| Concept | Tags | Entry | Summary | Confidence |
|---|---|---|---|---|
| dark mode toggle | ui, theme | `ui/theme/Toggle.kt:ThemeToggle` @ L12-L80 (class) | user-facing switch delegates persistence to ThemeStore. | high |
| dark mode toggle | ui, theme | `ui/theme/ThemeStore.kt:ThemeStore.setMode` (function) | applies theme change; writes to SettingsRepo. | high |
```

"Confidence" reflects whether the files have changed since `verified_at`, not whether the hint is correct for the current task. High-confidence still requires the agent to verify relevance.

The agent:
- Treats hints as **candidate starting points**, not truth
- Verifies relevance by reading
- Falls back to standard exploration if hints mislead
- Emits its own `### Code Map Entry` block (see Agent output contract above)

The agent MUST NOT read `code-map.jsonl` directly. The command is the sole reader and renders hints into the markdown table above.

## v1 → v2 migration

v1 (TSV, `code-map.md`) is fully deprecated. There is no automatic conversion: v1 entries are not ported.

On the first v2 write after the v2 release:

1. The command checks for `claude-output/_index/{repo-name}/code-map.md` (v1 file).
2. If present, the command presents a rename plan to the user via the Write Safety Gate:
   - source: `code-map.md`
   - destination: `code-map.v1.deprecated.md` (or `code-map.v1.deprecated.{ISO8601-UTC}.md` if the destination already exists)
3. On user approval, the rename is executed. On rejection, the v1 file is left untouched but `code-map.jsonl` still receives the new write (v1 and v2 coexistence is tolerated; only the v2 file is read).
4. After rename, the v1 file is never read or written again. Users may delete it at any time.

Subsequent v2 writes do not re-check for `code-map.md` (it is either renamed, deleted, or the user chose to keep it — all terminal states).

## Metrics Log

A side-effect append-only log at `claude-output/_index/{repo-name}/code-map-metrics.log` records each code-map operation for out-of-band analysis. **The log is never loaded into agent context** — it exists purely for user-driven inspection (grep/awk etc.). The log uses TSV (not JSONL) since humans read it directly.

### Format

```
# code-map-metrics v2
# cols: timestamp<TAB>op<TAB>counts
# op: read | write | gc
# timestamp: ISO 8601 UTC
# counts: comma-separated key:value pairs (no spaces)
2026-04-15T10:30:00Z	read	matched:5,verified:3,removed:0,passed:3,malformed:0
2026-04-15T10:31:12Z	write	appended:1
2026-04-15T11:00:00Z	read	matched:8,verified:7,removed:1,passed:7,malformed:0
2026-04-17T09:15:00Z	write	appended:0,reason:validation-failed
2026-04-17T09:15:30Z	write	appended:0,reason:no-block
```

### Write triggers

- **`read`** (after command-side read flow completes): `matched:{N},verified:{M},removed:{S},passed:{K},malformed:{X}` where N = keyword hits, M = passed verify, S = stale removals, K = records forwarded to agent, X = malformed/schema-invalid lines surfaced to user.
- **`write`** (after write pipeline completes, whether or not the append happened):
  - success: `appended:1`
  - skipped (no block / no starting points / not a git repo): `appended:0,reason:no-block` or `appended:0,reason:no-git-repo`
  - validation failed after retry: `appended:0,reason:validation-failed`

### Analysis (user-driven, out-of-band)

- Hit rate proxy: `passed / matched` over time
- Staleness rate: cumulative `removed` per unit time
- Validation failure rate: count of `reason:validation-failed` over time (signals prompt drift or schema mismatch)
- Warm-up curve: `passed` per read over time

The plugin writes but never reads this file. Log grows unbounded — users may truncate periodically if size becomes a concern.

## Scope limits (MVP)

- Single file per project (no module split until size becomes an issue — revisit at 1000+ entries)
- No cross-concept linking
- No team-shared index (individual `.claude-output/_index/` only; explicit promotion to CLAUDE.md is a human-driven operation, not a plugin feature)
- No concurrent-write coordination. Single-record appends ≤ 4096 bytes are empirically atomic on Linux (ext4/XFS) and macOS (APFS) via `O_APPEND`, but this is implementation-dependent rather than a POSIX guarantee (see "Format" section). Truly concurrent multi-session writes are out of scope; if one ever produces a malformed line, the reader's malformed-line handling surfaces it.
- No external JSON Schema validator integration. The command performs schema conformance via LLM-based evaluation of the Draft 2020-12 rules embedded in this document. This is a pragmatic trade-off — the self-check checklist, strict prompt examples, and retry-once-on-failure pipeline mitigate unreliability; external validator integration may be revisited if failure rates become problematic.
- No human-readable serialization. The file is AI-optimized JSONL; the plugin never renders it back to human form.
- **MVP parameter choices are empirical.** The following caps are not derived from cited research and are subject to revision based on operational signals (validation-failure rate, hit rate, prompt-drift events observed in the metrics log): `entries` 1-5, `summary` ≤ 179 chars + period, `aliases` ≤ 10 items (each ≤ 80 chars), `tags` ≤ 8 items (each ≤ 24 chars), `concept` ≤ 80 chars. Revisit once usage data accumulates.

## Versioning

This document describes **v2**. v1 (TSV, `code-map.md`) is deprecated and not interoperable. Future additive changes (new optional keys) remain v2 under Postel's law — readers ignore unknown keys, writers stay strict. Schema-breaking changes would require a v3.
