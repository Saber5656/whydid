# Title

`whydid hook record`: fail-silent ingest path

## Summary

Implement the hidden `whydid hook record` subcommand: read the command text from
stdin and scalar fields from flags, redact, and append one record — never
breaking the user's shell and never exceeding the latency budget.

## Context

This is the hot path executed at every prompt (DESIGN §5.4–5.5). Its contract is
absolute silence and speed (threat T8): all failures are swallowed, exit code is
always 0, and nothing is ever written to stdout/stderr.

## Scope

- `internal/cli` registration of `hook record` (hidden — not listed in help)
- `internal/hookrecord/hookrecord.go` (logic, unit-testable without the CLI)

## Detailed Requirements

1. Flags (all string/int, parsed with stdlib `flag`): `--exit-code` (int,
   required), `--start-ts`, `--end-ts` (float seconds; 0 allowed), `--shell`
   (`zsh|bash`), `--cwd`, `--tty`, `--tmux-pane` (may be empty). Unknown flags →
   silently ignore (forward compat with newer snippets), do not error.
2. Read stdin with a hard cap of 64 KiB (`io.LimitReader` of 64 KiB + 1 to
   detect overflow); overflow → truncate to 64 KiB and set `cmd_truncated=true`
   (DESIGN §5.5).
3. Skip silently (exit 0, no write) when any of: stdin empty/whitespace-only;
   first whitespace-separated token of the command is `whydid`;
   `WHYDID_SESSION_ID` unset or fails store sanitization; `WHYDID_DISABLE=1`.
4. Redact the command text via `redact.Apply` (built-ins + config custom
   patterns; if config load fails, proceed with built-ins only — never block the
   hook on config errors).
5. Build `store.Record` (`v:1`, seq handled by store) and `store.Append` under a
   100 ms `context` deadline covering the whole invocation; on deadline or any
   error: drop and exit 0.
6. Absolutely no output: guard by never writing to stdout/stderr in this code
   path (the CLI wrapper must not print usage errors for this subcommand either —
   parse failures also exit 0 silently).
7. Exit code is 0 on every path (DESIGN §5.5).

## Acceptance Criteria

- [ ] Feeding `printf 'aws s3 cp x s3://b --secret_key=AKIAAAAAAAAAAAAAAAAA'`
      results in a stored record whose `cmd` contains `[REDACTED:` and no raw key.
- [ ] 1 MiB of stdin → record stored with 64 KiB `cmd` and `cmd_truncated=true`.
- [ ] Each skip condition in requirement 3 produces exit 0 and no file change.
- [ ] With an unwritable state dir, exit 0, no output (T8/E8).
- [ ] `whydid help` output does NOT mention `hook`.
- [ ] Measured wall time for one invocation on CI < 100 ms; typical dev-machine
      run < 15 ms (log in PR description, not enforced in test).

## Validation

Unit tests invoking `hookrecord.Run` with fake stdin/env/tempdir; a
CLI-level test executing the built binary via `os/exec` asserting empty
stdout+stderr and exit 0 across all paths.

## Dependencies

05, 06.

## Non-goals

The shell snippets that call this (08/09); explain-side reading; any batching or
async spooling (known unknown U3 covers the escalation path).

## Design References

DESIGN.md §5.4–5.5, §6, §12.2 T8, §13 E7/E8; ADR-005.3 (redact-at-record).
