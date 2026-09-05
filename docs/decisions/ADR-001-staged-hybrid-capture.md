# ADR-001: Staged Hybrid Capture (hooks for metadata, opportunistic output capture)

Date: 2026-07-08
Status: Accepted (confirmed with product owner, 2026-07-08)

## Context

To explain why the previous command failed, whydid needs context about that command.
Candidate mechanisms, evaluated in docs/research/01-prior-art.md and
02-shell-integration-techniques.md:

1. **Metadata-only hooks** — record command line, exit code, cwd, timestamps via
   zsh/bash hooks. Cheap and universal, but the actual error message is unavailable.
2. **Always-on session recording** (`script`-style) — best fidelity, but continuously
   persists all terminal output (including secrets) to disk and is fragile
   (thefuck instant mode's documented issues).
3. **Re-running the failed command** to capture stderr — simple but re-executes
   side effects (deletes, deploys, billable calls). Categorically unsafe.
4. **Terminal-native capture** — read the screen content that already exists
   (tmux `capture-pane`), on demand, in memory only.

## Decision

Adopt a **staged hybrid**:

- A shell hook layer (zsh native hooks; bash via vendored bash-preexec) records
  per-command **metadata only**: command line (redacted before persistence),
  exit code, cwd, shell type, timestamps, tty, tmux pane. Nothing else is persisted.
- At `whydid` invocation time, if the session runs inside tmux and capture is
  enabled, the actual error output is captured **on demand** via
  `tmux capture-pane`, segmented to the failed command, redacted, used in memory,
  and never written to disk.
- The output-capture side is behind an `OutputProvider` interface so v2 can add
  terminal-emulator-specific providers (iTerm2, kitty, WezTerm) without touching
  the core.

whydid never re-runs the user's command to observe its output.

## Consequences

- Works in every terminal (metadata-only floor); quality upgrades automatically
  inside tmux.
- No continuous recording ⇒ no secrets-at-rest problem beyond the command strings
  themselves, which are redacted before persistence (see DESIGN.md §7).
- Explanation quality is environment-dependent; the UI must state which context
  was available (e.g. "error output not captured: not in tmux").
- Segmentation of pane content is heuristic; when it fails we degrade to
  metadata-only rather than sending mis-attributed text.
- Two implementation layers (hooks + capture) instead of one; reflected as separate
  issues (08, 09, 10) with independent test strategies.
