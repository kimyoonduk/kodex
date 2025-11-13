# Attribution Tracker Readiness Review

## Existing Git Snapshot & Undo Infrastructure
- `codex-rs/utils/git/src/ghost_commits.rs:22-152` defines the ghost-commit engine that snapshots the working tree by writing directly to a temporary index and invoking `git commit-tree`. It captures parent SHAs, honours optional force-includes, and records pre-existing untracked paths for clean restores.
- The helper uses a fixed “Codex Snapshot” identity (`codex-rs/utils/git/src/ghost_commits.rs:336-352`) and never updates refs, so commits are orphaned objects unless a caller records them.
- `GhostSnapshotTask` runs automatically at the start of each turn when the `ghost_commit` feature is enabled (`codex-rs/core/src/codex.rs:1781-1787`, `codex-rs/core/src/tasks/ghost_snapshot.rs:18-109`). The task captures a snapshot on a blocking thread pool and stores it as a `ResponseItem::GhostSnapshot`.
- Snapshots are persisted in the turn history but filtered out before prompts (`codex-rs/core/src/context_manager/history.rs:40-117`) and drive the undo stack (`codex-rs/core/src/tasks/undo.rs:61-115`) by restoring the last ghost commit.

## Turn-Level Diff Tracking
- `TurnDiffTracker` aggregates per-turn diffs in memory (`codex-rs/core/src/turn_diff_tracker.rs:25-250`). It records baselines on `apply_patch` begin events, keeps rename-aware UUID mappings, and emits unified diffs via `get_unified_diff`.
- Tool plumbing hooks the tracker around patch operations (`codex-rs/core/src/tools/events.rs:171-219`, `codex-rs/core/src/tools/events.rs:345-368`), so each apply_patch produces a `TurnDiff` SSE event.
- After the model finishes a turn, the tracker runs again to broadcast any remaining diff (`codex-rs/core/src/codex.rs:2165-2187`), ensuring the UI sees aggregated changes.

## Feature Flag & Configuration Surface
- The `ghost_commit` feature is registered as experimental (`codex-rs/core/src/features.rs:280-304`) and guards the snapshot task. Documentation still lists its default as `false` (`docs/config.md:43-53`), so behaviour/documentation drift exists.
- No dedicated flag currently controls attribution-specific behaviour; any new tracker will need its own toggle to avoid surprising users.

## Session Lifecycle & Data Sources
- Each turn records the incoming user message and the model’s outputs (`codex-rs/core/src/codex.rs:1763-2200`), giving us access to user prompts, tool calls, completion metadata, and token usage—raw material for commit messages or stats.
- Tool invocations are wrapped with `ToolInvocation` context (`codex-rs/core/src/tools/context.rs:18-74`), making it feasible to log which tools ran during a turn.

## What’s Missing for Attribution Commits
- Current ghost commits are intentionally detached; there is no branch management, ref updates, or author swapping—meaning there’s nothing to surface in `git log` without new work.
- The diff tracker outputs textual patches but never stages, commits, or reconciles them with ghost snapshots. There is no system that pairs “user turn vs Codex turn” changes.
- No code keeps a parallel ref/branch up to date. Background commits would need safe ref-locking, conflict handling, and eventual GC of ghost objects.
- There is no stat collection for “Codex vs user contribution” beyond raw tool telemetry. Any analytics will require new storage and presentation paths.

## Opportunities to Leverage
- Ghost commit creation already handles untracked files and repo-relative scopes; we can reuse it for turn baselines and as the source for Codex-authored commits.
- TurnDiffTracker’s rename-aware diff could seed commit content and avoid repeated `git status` calls.
- Undo history demonstrates how ghost commit IDs persist alongside conversation state, providing a ready-made place to remember the most recent Codex/user commit pair if needed.

## Risks & Gaps
- Documentation/config defaults conflict with runtime defaults, so enabling attribution must clarify configuration semantics.
- Commit authorship currently hard-codes a single identity. Supporting dual authors (user vs Codex) will require parameterisable author/committer settings and careful environment handling inside the sandbox.
- Detached ghost commits accumulate in the object database with no pruning strategy; introducing a long-lived branch makes GC less urgent but increases the need for ref housekeeping.
- The CLI wrapper (`codex-cli/bin/codex.js:1-109`) simply shells out to the Rust binary. Any attribution UX needs to remain transparent so the wrapper continues pointing at the main binary while hidden bookkeeping happens inside the Rust core.
