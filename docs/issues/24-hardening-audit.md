# Title

Hardening audit and error-path polish

## Summary

A checklist-driven pass over the integrated product that verifies and, where
needed, fixes the security-hygiene details that individual issues could not see
whole: permissions, error-message leakage, disable/kill switches, and umask
independence.

## Context

DESIGN threats T3/T5 assign their final "audit" mitigations here, after the
explain flow (16) and doctor (21) exist. This issue is deliberately a *bounded
checklist*, not an open-ended review: every item below is pass/fail.

## Scope

Small fixes across existing packages; no new features. Each fix references its
checklist item in the commit message.

## Detailed Requirements (the checklist — verify each, fix if failing)

1. **Umask independence**: with `umask 0077` and `umask 0000`, freshly created
   state dir, session files, and config are 0700/0600/0600 (explicit Chmod
   after create everywhere — grep for `os.Create`/`OpenFile`/`MkdirAll` and
   verify each site).
2. **Error-path leak review**: for every `fmt.Errorf`/message constant in
   `internal/llm`, `internal/explain`, `internal/consent`, `internal/doctor`:
   no request bodies, no URLs with query strings, no header values, no key
   material, no raw provider bodies beyond `SafeProviderMessage`. Produce a
   one-line-per-file review note in the PR.
3. **`WHYDID_DISABLE` completeness**: with it set — hooks no-op (08/09 tested),
   `hook record` skips (07 tested), `whydid` exits 2 (16 tested), and ALSO
   `doctor` still works (it must, for debugging) — add the missing doctor test.
4. **Signal handling**: SIGINT during S6 exits 3 with a clean single-line
   message and no goroutine leak (`goleak`-style hand-check via test with
   `runtime.NumGoroutine` before/after; no new dependency).
5. **Temp-file hygiene**: compaction (05) and config save (04) temp files are
   created 0600 in the same directory (no `/tmp`), and are removed on failure
   paths.
6. **stderr/stdout discipline sweep**: run every subcommand with stdout
   redirected to a file in a scripted test; assert stdout is empty for every
   command except: explain's fix emission, `init` (snippet), `config
   get/list/path` (values), `version`, and `help` — exactly the DESIGN §11.1
   contract.
7. **argv hygiene**: `ps`-visible argv of `whydid hook record` contains only
   the documented scalar flags — re-assert no command text in argv by
   inspecting the init snippets (08/09) once more after all changes.
8. **Provider request headers**: exactly the documented header sets (12/13) —
   no `User-Agent` customization leaking version + OS beyond Go's default?
   DECISION: set `User-Agent: whydid/<version>` explicitly (benign, useful for
   endpoint operators) and document it in DESIGN §10.6 via a PR that updates
   the docs file in the same change.
9. **go.mod audit**: runtime deps == {BurntSushi/toml}; test deps == {creack/pty};
   `go mod verify` clean; record in PR.
10. **golangci-lint severity pass**: zero `gosec` suppressions without an
    inline justification comment.

## Acceptance Criteria

- [ ] Every checklist item has: an automated test (new or existing, linked) OR
      a written verification note in the PR — items 1,3,4,5,6 require tests.
- [ ] The stdout-discipline test (item 6) is added to the permanent suite.
- [ ] DESIGN.md §10.6 updated with the User-Agent decision (item 8) in this PR.
- [ ] No behavior changes beyond the checklist fixes.

## Validation

CI green; PR contains the item-by-item audit table (pass / fixed-in-commit /
n-a with reason).

## Dependencies

16, 21.

## Non-goals

New features; refactors beyond fix-sized changes; dependency upgrades;
release wiring (25).

## Design References

DESIGN.md §12.2 T3/T5, §12.3, §11.1; ADR-004 (dependency policy), ADR-005.7.
