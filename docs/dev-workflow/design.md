# dev-workflow: Design Document

## Problem

dev-workflow is a Claude Code plugin for cases where explicit human oversight at each consequential step is preferred over end-to-end autonomous AI execution. AI-assisted coding tools typically optimize for code generation velocity; intermediate decisions (scope, investigation strategy, commit grouping) are often made implicitly by the AI. dev-workflow provides a structured alternative in which such decisions are surfaced for user approval.

## Design Principles

The following principles govern the plugin's behavior:

### 1. Approval at consequential decisions
Decisions with non-trivial cost (implementation plan, per-file changes, commit grouping, PR creation) require explicit user approval. The plugin exits and waits rather than inferring.

### 2. Read-only code investigation, separated from orchestration
Investigate agents (`task-implement:investigate`, `bugfix:investigate`) are configured with `tools: Glob, Grep, Read` and cannot modify files. Writes are performed by command-level orchestration. The tool restriction is enforced by the plugin system, not by convention.

### 3. File-based state
Workflow state persists as markdown files under `claude-output/{id}/`. Resume logic derives position from the files present. No session state, no external database.

### 4. Cache treated as a materialized view
Code-map navigation entries are derivative: every entry is verifiable against current code via path existence and `git diff` against the recorded `verified_at` commit. Stale entries are removed on read. Code remains the source of truth.

### 5. User approval as quality filter for cache writes
Cache entries are persisted only after the user has approved the investigation results. The approval that gates workflow progress is reused to gate cache quality.

### 6. Stated scope boundaries
The plugin does not auto-merge PRs, decide task scope without approval, modify `CLAUDE.md`, or proceed past unresolved 🔴 Required spec gaps.

## Key Design Decisions

Each decision notes what was chosen, the alternative considered, and trade-offs accepted.

### Markdown prompts as implementation

**Chosen**: Commands, skills, and agents are defined as markdown files interpreted by the LLM at runtime.

**Alternative**: Implementing logic in a programming language with structured APIs.

**Reasons**:
- Aligns with Claude Code platform conventions.
- Users can read the full instruction set.
- Workflows can be adapted per-project without rebuilding.
- The logic is primarily orchestration rather than computation.

**Trade-offs accepted**:
- Unit tests do not apply in the conventional sense. Verification relies on running workflows and inspecting state files.
- Behavior depends on LLM interpretation; structured output contracts and explicit numbered steps mitigate ambiguity.

### `shared/references/` directory for cross-skill contracts

**Chosen**: Protocol files referenced by multiple skills live in `plugins/dev-workflow/shared/references/`.

**Alternative**: Placing shared files under one skill's `references/` directory, with other skills referencing them cross-skill.

**Reasons**:
- A file used by multiple skills is not owned by any one skill.
- All consumers reference via the same path (`../../shared/references/...`).

**Trade-offs accepted**:
- Adds a directory layer. Useful only when ≥2 skills share a file.

### Narrow-door starting points, not exhaustive file list

**Chosen**: Each code-map entry records 1-3 file paths that serve as exploration entry points for a concept.

**Alternative**: Recording all files associated with a concept.

**Reasons**:
- The question the cache answers is "where should exploration start?", not "where do all related files live?".
- Fewer entries per concept reduce storage and simplify invalidation.
- Exhaustive listings may invite anchoring bias: future agents over-trust the listed files.

**Trade-offs accepted**:
- Requires the agent to select 1-3 entries during investigation.
- The cache does not expose the full file set for a concept; the agent must navigate from the starting points.

### LLM-based query expansion

**Chosen**: Keyword matching for cache reads uses LLM-generated semantic equivalents (synonyms, alternative namings) in addition to base keywords extracted from the task.

**Alternative**: Integrating a dedicated IR component (stemming, WordNet, embeddings).

**Reasons**:
- Uses the LLM already present in the runtime; no additional dependency.
- Can incorporate project-specific vocabulary from `CLAUDE.md`.
- Query expansion is a classical information retrieval technique; using an LLM as the expansion step is a modern instance.

**Trade-offs accepted**:
- Adds token usage per query for expansion.
- Expansion quality depends on LLM judgment, not a deterministic algorithm.

### Explicit target repository over heuristic inference

**Chosen**: Each workflow records the target git repository in `claude-output/{id}/meta.md` on first run. All subsequent git, gh, glab, and Bitbucket-API operations route through the recorded `target-repository-path`.

**Alternative**: Inferring target from cues at every invocation (ticket prefix, spec body keywords, file paths in earlier checkpoint files).

**Reasons**:
- Workspace layouts containing multiple sibling repositories make cue-based inference unreliable; ticket prefixes are not always repo identifiers, and spec text mentions multiple repos.
- One detection per `{id}` is cheap; persistent record means subsequent invocations have no ambiguity to resolve.
- Eliminates a class of "wrong repo" errors that are expensive to undo (branch created in wrong repo, commit pushed to wrong remote).

**Trade-offs accepted**:
- An extra file (`meta.md`) per workflow.
- Detection requires user interaction when multiple candidate repos exist at or directly below `$PWD`.
- No automated migration for existing checkpoints without `meta.md`; the first re-run after upgrade runs detection.

### Metrics log outside agent context

**Chosen**: Code-map operations append to `code-map-metrics.log`. The log is never loaded into any agent's context. Analysis is performed by the user via shell tools.

**Alternative**: Loading metrics into agent context, or exposing analysis via a built-in command.

**Reasons**:
- Observability without additional token cost.
- Avoids the possibility of agents acting on their own metrics.
- Log format is plain-text, durable, inspectable.

**Trade-offs accepted**:
- No built-in analysis UI.
- Log grows unbounded; manual rotation is expected.

## Current Scope (v1.9.0)

### Included
- Feature development flow: `spec-review` → `spec-breakdown` → `task-implement`
- Bug fix flow: `bugfix` (internally invokes `spec-breakdown` and `task-implement`)
- Two read-only investigate agents
- Code-map navigation index:
  - Query expansion for keyword matching
  - Multi-layer invalidation (path existence, `git diff` against `verified_at`, commit reachability)
  - User-facing summaries at read/write touchpoints
  - Append-only metrics log
- Multi-repo workspace support (target repository explicitly recorded per-`{id}` in `claude-output/{id}/meta.md`; all repository-bound CLI operations route through the recorded `target-repository-path`)
- `shared/references/` protocol layer

### Not included
- Automated execution without user approval at consequential steps
- Spec drift detection (change detection on `source.md` during implementation)
- Post-implementation verification agent
- Batch scheduler or multi-workflow dashboard
- Team-shared code-map
- Symbol-level or module-level indexing
- Automatic test harness integration
- Non-git repositories (git is required)

## Deferred Features

The following are candidates for future iteration. Order is tentative and will be revised based on observed operational data.

### Candidates aligned with current design
- Spec drift detection: detect whether `source.md` has been modified during implementation.
- Structured `plan.md`: machine-parseable task format as a precondition for downstream tooling.
- Scheduler / batch command: tracks multiple workflows, suggests next actions, relies on existing idempotent commands.

### Candidates dependent on accumulated data
- Task dependency inference from overlap of past starting points.
- Symbol-level or module-level indexing if concept-level hit rate plateaus.

### Candidates requiring further design
- Post-implementation verification agent (read-only review against spec intent).
- Domain-specialized investigate agents (security, performance, accessibility).
- Multi-model routing for different steps.
- Team-shared code-map (manual promotion to `CLAUDE.md` remains the user-driven path).

## Measured Outcomes

At time of writing, the plugin has been implemented but not yet operated in production. The following metrics are planned for observation via `code-map-metrics.log`; no results are reported yet.

### Metrics to observe
- **Hit rate** = `passed / matched` per read operation.
- **Staleness rate** = cumulative `removed` entries per unit time.
- **Warm-up curve** = `passed` per read over time.
- **Growth rate** = `appended` events per unit time.

### Decisions planned based on data
- If hit rate is consistently low relative to the added complexity, reconsider concept derivation and expansion logic.
- If staleness rate is high relative to write rate, reconsider invalidation strategy.
- If hit rate is consistently usable, consider proceeding to items in Deferred Features.

Specific thresholds are not stated here: hit rate, staleness, and warm-up behavior depend heavily on project size, change velocity, and task granularity. Thresholds will be determined empirically per-project.

## References

### Project files
- `README.md` — installation and usage.
- `plugins/dev-workflow/.claude-plugin/plugin.json` — manifest.
- `plugins/dev-workflow/commands/*/command.md` — orchestration entry points.
- `plugins/dev-workflow/agents/*.md` — investigate agent definitions.
- `plugins/dev-workflow/shared/references/code-map-format.md` — code-map protocol.

### External concepts
- Query expansion (information retrieval): enriching a query with semantically related terms before matching. Origins in classical IR literature (Rocchio, 1971, and subsequent).
- Materialized view (databases): a precomputed view derived from base tables and invalidated when base changes. Used here as a metaphor for the code-map cache, with code as the base.
