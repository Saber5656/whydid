# Title

Shell end-to-end PTY test harness

## Summary

Build the PTY-based e2e harness that proves the full product loop in real
shells: init → failing command → record → explain against a local fake provider
→ selection → fix lands in the zsh buffer / bash history.

## Context

Unit layers can't prove the shell-side halves (hooks firing, wrapper capture,
`print -z`, `history -s`). DESIGN §14 mandates this harness; it is also the
system-level proof of INV-2 and T8.

## Scope

- `e2e/` package with build tag `e2e` (runs in CI via `make test-e2e`; included
  in the CI workflow from this issue on)
- Test-only dependency `github.com/creack/pty` (allowed by ADR-004)
- A tiny in-test fake LLM HTTP server (reuses issue 12's fake shapes) started
  on 127.0.0.1 with canned §10.4 responses

## Detailed Requirements

1. Harness primitives: spawn `zsh -i` / `bash -i` under a PTY with a controlled
   env (`HOME=tempdir`, `WHYDID_STATE_DIR`, `WHYDID_CONFIG` pointing at a
   prepared config with `provider=openai`, `base_url=http://127.0.0.1:<port>/v1`,
   `model=test`, consent pre-granted, `WHYDID_API_KEY=dummy`); helpers
   `send(line string)`, `expect(regex, timeout)` over the PTY stream.
2. Scenarios (each for BOTH shells unless noted):
   a. **Record**: `eval "$(whydid init <shell>)"`, run `ls /nonexistent`,
      assert session file contains one record `exit != 0` within 2 s.
   b. **Explain+insert (zsh)**: run scenario-a, then `whydid`, `expect` the
      fixes list, send `1`, then assert the ZLE buffer pre-fill by sending a
      newline and expecting the fake fix command's echo as the next executed
      command line (the canned fix is `echo WHYDID_E2E_OK`, so `expect
      WHYDID_E2E_OK`).
   c. **Explain+history (bash)**: as (b) but after selection send `\x1b[A`
      (up-arrow) + newline; expect `WHYDID_E2E_OK`.
   d. **Nothing to explain**: after a succeeding command only → `whydid` exits 1
      with the documented message.
   e. **Hook resilience (T8)**: point `WHYDID_STATE_DIR` at a read-only dir;
      run 3 commands; prompt remains functional (send/expect echo markers);
      no error text on the PTY.
   f. **INV-2 at system level**: canned fix includes UTF-8 + `$HOME` literal;
      the inserted line (read back from the buffer echo) is byte-identical.
   g. **Latency guard (informative)**: time 20 no-op prompts with hooks on vs
      off; log the delta; assert only a generous bound (< 100 ms mean) to stay
      non-flaky (G1's 15 ms is a dev-machine target, not a CI assertion).
3. Skip logic: scenarios require the shell binary present; zsh is installed on
   CI ubuntu (issue 02) and present on macOS; bash everywhere. `t.Skip` with a
   loud message otherwise.
4. Deterministic canned responses; no network beyond 127.0.0.1; total suite
   < 120 s.

## Acceptance Criteria

- [ ] All scenarios pass on ubuntu-latest and macos-latest in CI.
- [ ] Harness failure output includes the last 50 PTY lines (debuggability).
- [ ] `make test-e2e` runs locally with only Go + shells installed.
- [ ] go.mod gains only `creack/pty`, marked test-only in a comment.

## Validation

CI matrix; a recorded local run in the PR description. Flake policy: any flake
in the first week → issue to deflake before wave 6 proceeds.

## Dependencies

08, 09, 12, 16, 18.

## Non-goals

tmux-inside-PTY capture e2e (tmux behavior is unit-tested in 10 and dogfooded —
U1); Windows; performance benchmarking beyond the guard.

## Design References

DESIGN.md §14 (shell e2e row), §12.2 T8, §12.3 INV-2, §5; ADR-003, ADR-004.
