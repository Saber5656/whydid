# Title

Text hygiene: redaction engine and ANSI sanitizer

## Summary

Implement the two shared text-hygiene packages: `internal/redact` (mandatory
secret masking per ADR-005) and `internal/sanitize` (ANSI/control-sequence
stripping). Both ship with golden-file corpora.

## Context

Redaction runs at record time and payload time (DESIGN §7.1) and is the primary
mitigation for threats T2/T3. Sanitization defends T1/T4 and is shared by
capture (10), parser (15), and renderer (17).

## Scope

- `internal/redact/rules.go` — built-in rule table (DESIGN §7.2), compiled once
- `internal/redact/redact.go` — `Apply(s string, custom []string) string`
- `internal/redact/testdata/corpus/*.txt` + golden outputs
- `internal/sanitize/sanitize.go` — `Strip(s string) string`
- `internal/sanitize/testdata/` corpus

## Detailed Requirements

1. Implement every rule row of DESIGN §7.2 with the stated IDs, application
   order, and replacement forms (`[REDACTED:<id>]`, group-preserving where the
   table says so). Rules compile at init into a package-level slice; `Apply`
   runs them in order, then the caller-supplied custom patterns (replacement
   `[REDACTED:custom]`).
2. Custom patterns arrive pre-validated (issue 04 validates compilability), but
   `Apply` must not panic on a bad pattern — compile errors → skip that pattern.
3. Performance bound: 1 MiB input through all built-ins in < 50 ms on CI
   hardware (DESIGN §7.4) — add a benchmark and a non-flaky sanity test with a
   generous ceiling (e.g. < 500 ms) to catch pathological regressions.
4. `sanitize.Strip`: remove ESC-introduced sequences — CSI (`ESC [ … final`),
   OSC (`ESC ] … BEL or ESC \`), DCS/SOS/PM/APC (`ESC P/X/^/_ … ESC \`), single
   ESC+byte sequences — and all C0 controls except `\n` and `\t`; also strip
   `\r`. Must terminate on unterminated sequences (strip to end of string).
   Valid UTF-8 in, valid UTF-8 out; multibyte text untouched.
5. Both functions are pure (no I/O, no logging, no globals beyond compiled
   rules).
6. Required corpus cases (DESIGN §7.4): every rule positive; false-positive
   guards — 40-char git SHA, UUIDv4, `--token-budget` flag name, base64 filename
   argument — must pass through unchanged; multi-hit line; overlapping matches;
   multi-line PEM block; for sanitize — color SGR, cursor movement, OSC-8
   hyperlink, title-set OSC-0, raw BEL, an unterminated CSI at EOF.

## Acceptance Criteria

- [ ] Golden tests cover every DESIGN §7.2 rule ID and every listed
      false-positive guard.
- [ ] `Apply` output for the corpus matches goldens byte-for-byte.
- [ ] `Strip` corpus: no ESC (0x1B), no C0 except `\n`/`\t` in any output.
- [ ] Benchmark exists; sanity test enforces the generous ceiling.
- [ ] Package docs state the ADR-005 rule: built-ins are not user-disableable.

## Validation

`go test ./internal/redact/... ./internal/sanitize/...`; goldens reviewed by a
human in the PR (they are the de-facto redaction spec).

## Dependencies

01.

## Non-goals

Where redaction/sanitization are invoked (07/10/14/15/17 own call sites);
entropy-based detection (v2); locale-aware anything.

## Design References

DESIGN.md §7, §9.3 (sanitize contract), §12.2 T1–T3; ADR-005.
