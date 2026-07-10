# Title

First-run consent flow and `--show-payload`

## Summary

Implement `internal/consent` (the one-time consent gate per ADR-005.4/DESIGN
§2.3) and the `--show-payload` behavior that prints the exact outbound payload
without any network call.

## Context

Consent is state S2 of the explain flow and must hard-block any network I/O
until accepted (INV-3). `--show-payload` is the inspectability commitment
(ADR-005.5) and is also reachable from the selection loop's `s` key (18).

## Scope

- `internal/consent/consent.go` — `CurrentVersion` const (= 1),
  `Required(cfg) bool`, `Run(tty io.ReadWriter, cfg *config.Config,
  endpointHost, provider string) error`
- `internal/payloadpreview/preview.go` (or a function in consent pkg) —
  `ShowPayload(w io.Writer, p payload.Payload, prompts llm.Request)`

## Detailed Requirements

1. `Required` = `cfg.Privacy.ConsentVersion < CurrentVersion`.
2. `Run` renders the DESIGN §2.3 text verbatim-equivalent (fill in endpoint
   host and provider; the field list and "never" list are fixed strings that
   must match ADR-005.1's payload scope), prompts `Proceed? [y/N] `, reads one
   line from tty: `y`/`Y`/`yes` → set `cfg.Privacy.ConsentVersion =
   CurrentVersion`, `config.Save`, return nil; anything else → return
   `ErrDeclined` (caller exits 2, DESIGN §2.3).
3. No-TTY case is decided by the caller (16): consent required + no tty →
   exit 2 with "run `whydid` interactively once to review what is sent"
   (DESIGN §2.4) — provide the message constant here.
4. `ShowPayload` prints to the given writer (stderr in S5; tty in the `s` key
   path): a header `payload that would be sent to <host>:`, the pretty payload
   JSON, a separator, then the system prompt and user prompt labeled — all
   passed through `sanitize.Strip` defensively. Never prints key material
   (payload/prompts contain none by construction — assert in test).
5. Endpoint host shown = hostname of the resolved provider base URL (helper
   from issue 11's URL parsing; do not re-derive ad hoc).
6. Bump procedure documented in the package comment: changing consent-relevant
   behavior requires bumping `CurrentVersion` (re-consent) + updating ADR-005.

## Acceptance Criteria

- [ ] Scripted tty tests: `y` persists `consent_version=1` (config file
      re-loaded shows it) and returns nil; `n`/empty → `ErrDeclined`, config
      unchanged.
- [ ] Consent text golden test; contains the literal lines for all 6 payload
      fields and the four "never" items (history, environment variables, file
      contents, key material).
- [ ] `ShowPayload` golden test for a tmux-case payload: contains payload JSON
      + both prompts; no ESC bytes after an escape-containing input.
- [ ] `Required` false after acceptance (idempotent second run skips the flow).

## Validation

`go test ./internal/consent/...` with temp config homes and scripted tty
doubles.

## Dependencies

04, 14.

## Non-goals

Wiring into the explain state machine (16); telemetry of consent (none, ever);
localizing the consent text (English fixed in v1 — `ui.language` affects only
LLM output fields, DESIGN §10.2).

## Design References

DESIGN.md §2.3–2.4, §9.1 S2/S5, §12.3 INV-3; ADR-005.4–5.
