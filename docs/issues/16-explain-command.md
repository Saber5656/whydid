# Title

Explain command orchestration (state machine S0–S10)

## Summary

Implement the default `whydid` command: the S0–S10 state machine of DESIGN §9.1
wiring config, store, consent, capture, payload, provider, parser, renderer,
and selection into the end-to-end explain flow with the exact exit-code
contract.

## Context

This is the integration point of waves 1–4. Everything it calls already exists;
this issue owns sequencing, error-to-exit-code mapping, the context-used line,
the spinner, and non-interactive degradation.

## Scope

- `internal/explain/explain.go` — `Run(deps Deps, flags Flags) int`
- `internal/explain/deps.go` — `Deps` struct bundling interfaces (store reader,
  capture provider, llm factory, consent, tty opener, stdout/stderr writers,
  now-func) so every state is testable with fakes
- `internal/cli` wiring: default command + flags `--show-payload`,
  `--no-capture`
- Minimal spinner (`internal/explain/spinner.go`): dots on stderr while the
  provider call is in flight, only when stderr is a TTY; stops cleanly

## Detailed Requirements

1. Implement S0–S10 exactly as DESIGN §9.1, including:
   - S0: `WHYDID_DISABLE=1` → stderr notice, exit 2. `config.Load`; validation
     error → exit 2 with the key-specific message; print load warnings (unknown
     keys) once to stderr.
   - S1: `WHYDID_SESSION_ID` unset → exit 1 with init/doctor hint; session file
     missing/empty → exit 1 ("no records…"); newest relevant record succeeded →
     exit 1 ("last command exited 0 — nothing to explain"); target selection via
     `store.LatestFailure(records, 10)`.
   - S2: `consent.Required` → if no tty, exit 2 with the issue-19 message const;
     else run flow; declined → exit 2.
   - S3: skip when `--no-capture` or `context.capture == "off"` (Result
     Source:"none", FailReason "capture disabled"); otherwise call provider.
   - S4: `payload.Build` + `payload.Prompts(cfg.UI.Language)`;
     `config.RequireModel()` error → exit 2 with "set llm.model" message (E9).
   - S5: `--show-payload` → `ShowPayload` to stderr, exit 0, and assert no
     provider was constructed (INV-3 test hook).
   - S6: `llm.New(cfg)` + `Explain` under `context.WithTimeout(cfg timeout)`;
     SIGINT during the call cancels the context and exits 3 ("canceled");
     error taxonomy → exit mapping: `E_NO_KEY`/`E_INSECURE_ENDPOINT` → 2, all
     other llm errors → 3, each with its one-line message.
   - S7: `parse.Parse`; `ErrMalformed` → exit 4, and print (stderr) the S1
     metadata summary ("command, exit code, duration") so the user still gets
     something (DESIGN §9.1 S7).
   - S8: compose the context-used line: `command, exit code N` +
     `, error output (tmux)` when captured, or `; error output: not captured
     (<FailReason>)` — then `render.Explanation`.
   - S9/S10: tty open success → `Select`; chosen → `EmitFix` to stdout; nil →
     empty stdout; both exit 0. tty open failure → non-interactive mode
     (DESIGN §9.4): render only, empty stdout, exit 0.
2. Terminal width for the renderer: from the tty when available (TIOCGWINSZ via
   `golang.org/x/term`? NO — no new dependency: use a plain ioctl via
   `syscall`/`unsafe` OR accept 80 as fixed default; DECISION: fixed 80 when
   width cannot be read via stty-free means; keep it simple, note in code).
3. Spinner only between request start and response; suppressed when stderr is
   not a TTY or `NO_COLOR` set; never appears in captured outputs.
4. Every stderr message in this flow is a single actionable line (constants
   collected in one file `messages.go` for reviewability).
5. Session GC (`store.GC`, 72 h) runs opportunistically at the end of any
   invocation that got past S1, errors ignored (DESIGN §6.3).

## Acceptance Criteria

- [ ] A full-fake test drives the happy path: seeded store + fake capture +
      httptest provider + scripted tty → stdout == chosen fix + `\n`, exit 0.
- [ ] Exit-code matrix test covering every row: 1 (no session, no records,
      newest-succeeded), 2 (disable, config invalid, no consent+no-tty,
      declined, no model, no key, insecure endpoint), 3 (network, timeout,
      provider 5xx, SIGINT), 4 (malformed), 0 (happy, quit, non-interactive,
      show-payload).
- [ ] `--show-payload` performs zero HTTP requests (fake asserts).
- [ ] `--no-capture` yields the "capture disabled" context line.
- [ ] Non-interactive mode (tty opener returns error): renders to stderr,
      stdout empty, exit 0.
- [ ] GC invoked once per run (fake clock/dir assertion).

## Validation

`go test ./internal/explain/...` with the Deps fakes (this is the biggest test
surface in the repo — table-driven by state); full-binary smoke happens in
issue 23.

## Dependencies

10, 11, 12, 13, 14, 15, 17, 18, 19.

## Non-goals

`hook record` (07), doctor (21), any auto-retry (U4), auto-invocation modes
(v2), reading records from other sessions.

## Design References

DESIGN.md §2.4, §9 (all), §10.6, §13 E9; ADR-003, ADR-005; ISSUE_PLAN §3.
