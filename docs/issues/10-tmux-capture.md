# Title

tmux OutputProvider: on-demand error-output capture

## Summary

Implement `internal/capture` with the `OutputProvider` interface and the tmux
provider: capture the failed command's actual output from the pane scrollback at
explain time, segment it, sanitize + redact it, in memory only.

## Context

This is the "staged" half of ADR-001: quality upgrade when available, honest
degradation otherwise. Unattributed text must never be sent (DESIGN §8.2.4);
captured bytes are hostile (T1) and may contain secrets (T2).

## Scope

- `internal/capture/capture.go` — `Result`, `OutputProvider` (DESIGN §8.1)
- `internal/capture/tmux.go` — the tmux implementation
- `internal/capture/segment.go` — segmentation heuristic (pure function,
  heavily unit-tested)

## Detailed Requirements

1. Types exactly per DESIGN §8.1 (`Result{Text, Source, FailReason}`;
   `Source ∈ {"tmux","none"}`).
2. Precondition ladder (each miss → `Source:"none"` with the specific
   `FailReason` strings): capture disabled by config/flag; record has empty
   `tmux_pane`; `TMUX` env unset; `tmux` binary not in PATH.
3. Invocation: `tmux capture-pane -p -J -t <pane> -S -<N>` where
   `N = min(error_output_lines*4, 2000)`; run via `exec.CommandContext` with a
   2 s timeout; non-zero exit or timeout → `Source:"none"`,
   `FailReason:"tmux capture failed"` (E13).
4. Segmentation (`segment.Extract(capture string, cmd string, maxLines, maxChars
   int) (string, bool)`), per DESIGN §8.2.3:
   - needle = first line of the record's command, space-runs collapsed, trimmed;
   - scan captured lines bottom-up for the last line whose space-collapsed form
     contains the needle;
   - segment = lines strictly after the match; drop trailing empty lines; drop
     the final line if it equals the last non-empty captured line's prefix
     (current-prompt heuristic as written in DESIGN);
   - keep the last `maxLines` lines and last `maxChars` chars (tail-biased),
     prepending `…[truncated by whydid]` when cut;
   - not found → `("", false)` → provider returns
     `FailReason:"capture could not be attributed"`.
5. Post-processing order: segment → `sanitize.Strip` → `redact.Apply` →
   `Result.Text` (DESIGN §8.2.5–6).
6. INV-4: no file writes anywhere in this package (enforced by review + the
   issue-22 audit).
7. Everything except the `exec` call must be pure/unit-testable; the tmux
   runner is injected as `func(ctx, args...) (string, error)` for tests.

## Acceptance Criteria

- [ ] Table-driven segmentation tests: simple case; multi-line command (match on
      first line only, E1); command string appearing twice (newest match wins);
      needle absent → not-found; prompt-line trimming; wrapped lines joined by
      `-J` (fixture reflects joined input); truncation marker exact.
- [ ] Precondition ladder: each rung returns the documented FailReason.
- [ ] Fixture with ANSI colors + an OSC sequence + a fake AWS key yields Text
      with no ESC bytes and `[REDACTED:aws-key-id]`.
- [ ] Injected runner proves the exact tmux argv (`-p -J -t %3 -S -160` for
      default config).
- [ ] Timeout path returns within ~2 s and degrades correctly.

## Validation

`go test ./internal/capture/...` (no real tmux needed — injected runner);
optional manual smoke inside tmux documented in the PR description. Real-tmux
behavior is revisited by dogfooding (known unknown U1).

## Dependencies

04, 05, 06.

## Non-goals

Other terminal providers (v2); persisting captures (forbidden, INV-4); prompt
markers or shell-side assistance (U1's possible follow-up).

## Design References

DESIGN.md §8, §12.2 T1/T2, §13 E1/E13; ADR-001; research/02 §5.
