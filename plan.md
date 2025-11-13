# Attribution Tracker Implementation Plan

## Objectives
- Maintain an invisible “chat history” branch that records two commits per turn (Codex-authored and user-authored) without interrupting the working tree.
- Capture rich metadata (user prompt, models, tools, token usage, optional billing) in commit messages for later analysis.
- Keep the feature fully gated so existing users observe no behaviour changes until they opt in.
- Provide a path to collapse the chat branch into production-ready commits while retaining distinct authorship.

## Proposed Architecture
1. **Attribution feature flag** – Introduce a dedicated toggle (e.g. `attribution_commits`) alongside the existing `ghost_commit` flag to scope the behaviour (`codex-rs/core/src/features.rs`, `docs/config.md`).
2. **Hidden branch manager** – Build a new component in `codex_git` that can create/update a ref such as `refs/codex/chat/<session-id>` using plumbing commands (extend `utils/git/src/operations.rs` alongside the existing snapshot helpers at `codex-rs/utils/git/src/ghost_commits.rs:22-152`).
3. **Turn lifecycle hook** – Augment the turn runner (`codex-rs/core/src/codex.rs:1763-2187`) to:
   - On turn start: diff working tree against last Codex commit, create a user-authored commit on the chat branch if changes exist.
   - On turn completion: package Codex changes (using `TurnDiffTracker`, `codex-rs/core/src/turn_diff_tracker.rs:25-250`) into a Codex-authored commit, referencing the newly created user commit as parent.
4. **Author-aware ghost commits** – Extend `CreateGhostCommitOptions` with optional author/committer overrides so snapshots can record either the user’s git identity (from repo/global git config) or the existing Codex identity without altering `GhostCommit` consumers. Fall back to a placeholder such as `codex-user <codex-user@codex.local>` when no git config is present.
5. **Commit assembly** – Reuse ghost snapshots as the staging area: capture a ghost commit, then update the chat branch ref to point at the new commit while preserving the ghost tree. Author overrides flow via the new options.
6. **Metadata capture** – Collect message text, tool invocations (`codex-rs/core/src/tools/context.rs:18-74`), model identifiers, token usage, and optionally cost to embed in commit messages (structured format, e.g. YAML front-matter).
7. **Branch squash workflow** – Design a reconciliation command that can collapse the alternating chat commits into two commits (user + Codex) without losing authorship. Likely approach: replay the chat branch onto a temporary branch, then use tree-level reconstruction (not `git merge --squash`, which drops signatures) to produce final commits while preserving distinct authors.
8. **Stats surface** – Expose summary data (e.g. percent Codex lines) via a new CLI subcommand or TUI panel, backed by the chat branch history.
9. **Documentation & migration** – Update configuration docs, onboarding, and possibly `docs/release_management.md` to explain the new attribution mode and squashing workflow.

## Work Breakdown
1. **Feature gating & config**
   - Add `Feature::AttributionCommits` and ensure it is disabled by default.
   - Document the flag in `docs/config.md` and sample configs.
2. **Git plumbing enhancements**
   - Add author override support to `CreateGhostCommitOptions`, defaulting to current behaviour when unset.
   - Extend `codex_git` with ref-reading/writing helpers for the hidden branch.
   - Implement safety checks for detached HEAD, rebase in progress, or non-fast-forward updates.
3. **Session integration**
   - Track the latest chat-branch commit ID inside session state (parallel to how ghost commits are stored in history).
   - Ensure branch updates run in blocking tasks and respect the sandbox constraints.
4. **Commit message generator**
   - Create a formatter that pulls data from `TurnDiffTracker`, tool telemetry, and response metadata to produce human-readable commit messages.
   - Include guardrails for large messages and privacy-sensitive content.
5. **Squash/Export tooling**
   - Provide commands to materialise the paired commits into the main branch (e.g. `codex export-attribution --target main`).
   - Implement algorithms to interleave / merge user and Codex hunks while keeping authors distinct.
6. **Statistics module**
   - Parse the chat branch history to compute metrics (commit counts, LOC by author, survival rate) and surface them in CLI/TUI.
7. **Testing & validation**
   - Unit-test new git helpers with temp repositories (pattern already established in `codex-rs/utils/git/src/ghost_commits.rs:360-419`).
   - Integration-test turn flows via the existing task suites, ensuring commits appear with correct authors and metadata.
   - Add regression tests for squashing/export commands.
8. **Telemetry & observability**
   - Emit tracing around branch updates and failures to aid debugging.
   - Optionally add metrics for commit latency or failure counts.

## Opportunities
- Leverage ghost commits for consistent tree snapshots and restoration without touching the working index (`codex-rs/utils/git/src/ghost_commits.rs:67-169`).
- Reuse `TurnDiffTracker`’s rename-aware diffs to avoid re-running `git diff` (`codex-rs/core/src/turn_diff_tracker.rs:25-250`).
- Build on the existing turn events that already collect token usage and tool metadata (`codex-rs/core/src/codex.rs:1763-2200`).

## Challenges & Open Questions
- **Authorship switching** – Resolve identity via repo `user.name`/`user.email`, then global config; otherwise use the placeholder author. Ensure behaviour is clearly messaged when the placeholder is applied.
- **Branch safety** – Hidden branch updates must be resilient to rewrites, detached HEADs, or repository resets. Need clear failure handling and user notifications.
- **Performance** – Running extra git plumbing twice per turn could be expensive on large repositories; caching tree objects via ghost commits should mitigate but must be measured.
- **Metadata size** – Commit messages may balloon if they include full prompts or diff stats; we need truncation policies and perhaps structured trailers for machine parsing.
- **Squash semantics** – Designing a deterministic algorithm to merge interleaved Codex/user commits while preserving blame attribution is non-trivial; may require custom tooling or staged replays.
- **User transparency** – Hidden branch work must remain invisible unless the feature is enabled; we need guardrails so standard git commands do not show the branch unexpectedly (e.g. use `refs/codex/...` namespace rather than `refs/heads`).

## Validation & Rollout Strategy
- Ship behind the new feature flag and require explicit opt-in.
- Provide a migration command to clean up or delete the chat branch if users disable the feature.
- Gather telemetry via tracing/metrics before making the feature generally available.
- Update release notes and documentation once the feature is stable.
