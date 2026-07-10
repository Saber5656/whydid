# Title

`whydid doctor` and session GC

## Summary

Implement `whydid doctor`: the ten-point environment diagnosis of DESIGN §11.5
with ✓/✗/– table output, one remediation line per failure, and opportunistic
session GC.

## Context

Doctor is the primary support tool (bug template asks for its output, issue 03)
and the user-visible verification step after install (DESIGN §2.1). It must be
safe to paste publicly: no key values, and record summaries are already
redacted at rest.

## Scope

- `internal/doctor/doctor.go` — `Run(deps Deps, w io.Writer) int` with
  injectable deps (config loader, env, store, key resolver, dialer, exec
  lookup, rc-file reader)
- `internal/cli` wiring: `whydid doctor`

## Detailed Requirements

1. Checks, in this order, each producing `✓` (pass), `✗` (fail, with one
   remediation line), or `–` (not applicable), exactly the DESIGN §11.5 list:
   1. binary path (`os.Executable`) + version string;
   2. `WHYDID_SESSION_ID` present → hook active this shell; absent → ✗ with
      `eval "$(whydid init <shell>)"` hint;
   3. rc-file grep (best-effort): `~/.zshrc` for zsh users, `~/.bashrc` and
      `~/.bash_profile` for bash — looks for the literal substring
      `whydid init`; unreadable files → `–`;
   4. state dir exists, dir mode 0700, session files 0600 (report worst
      offender; no auto-fix in v1);
   5. config parses; warnings listed; provider/model set (`RequireModel`);
   6. key resolvable → print SOURCE only (issue-11 sourceDescription); `E_NO_KEY`
      → ✗ with per-provider env-var hint;
   7. endpoint dial: TCP+TLS handshake (or plain TCP for loopback http) to the
      resolved host:port with 3 s timeout — no HTTP request is sent (zero
      cost, INV-3-compatible since consent covers LLM sends, and this sends no
      payload; still gate it behind consent? NO — a TLS handshake carries no
      user data; document this reasoning in a code comment);
   8. tmux: binary found + `$TMUX` set (both reported separately, `–` when
      absent by design);
   9. session file exists for current session; print last-record summary:
      `last: exit=<n> <first 60 chars of cmd>` (cmd is stored post-redaction);
   10. run `store.GC(72h)` and report `removed N stale session file(s)`.
2. Exit code: 0 when no ✗ (– allowed), else 1 (DESIGN §11.5).
3. Output is aligned plain text (no color in v1 — doctor output gets pasted
   into issues), stable ordering, one check per line, remediation lines
   indented beneath their check.
4. Never prints: key values, full commands > 60 chars, absolute HOME paths in
   remediations (use `~` where possible).

## Acceptance Criteria

- [ ] Fake-deps table test: all-green environment → exit 0 and 10 ✓/– lines.
- [ ] Each degraded scenario (no session id, bad perms, missing model, no key,
      unreachable endpoint, no tmux) flips exactly its own line to ✗/– with the
      documented remediation, exit 1 where ✗.
- [ ] Key line shows `env:OPENAI_API_KEY`-style source, never a value (test
      plants a key value and asserts absence in output).
- [ ] GC line reports the number removed; stale fixture actually deleted.
- [ ] Output golden test (all-green case) for format stability.

## Validation

`go test ./internal/doctor/...` with injected fakes; manual run pasted into the
PR description from a real machine.

## Dependencies

04, 05, 08, 09, 11.

## Non-goals

Auto-fixing problems (perms/rc edits — v2); network HTTP test calls; checking
provider quota/validity of the key (would cost money).

## Design References

DESIGN.md §11.5, §2.1, §6.3, §12.2 T3/T5.
