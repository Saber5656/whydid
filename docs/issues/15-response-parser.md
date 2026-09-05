# Title

Response parser, validator, and danger tagging

## Summary

Implement `internal/parse`: turn raw model text into a validated, sanitized
`Result{Explanation, Fixes, Confidence}` per DESIGN §10.4, including the
defensive JSON extraction, all field constraints, and the danger-tag overlay.

## Context

Model output is untrusted (T4). This package produces the single sanitized
struct that both the renderer (17) and stdout emission (18) consume — INV-2
holds by construction because both read the same object.

## Scope

- `internal/parse/parse.go` — `Parse(raw string) (Result, error)`
- `internal/parse/danger.go` — `dangerPatterns` list + `applyDangerTags`
- `internal/parse/testdata/` adversarial corpus

## Detailed Requirements

1. Types:

   ```go
   type Fix struct { Command, Description, Risk string }
   type Result struct { Explanation string; Fixes []Fix; Confidence string }
   ```

2. Extraction (DESIGN §10.4.1): trim; if the text contains a fenced block
   (``` with optional language tag), use the first fence's inner text; else take
   substring from the first `{` to the last `}`; if neither yields text starting
   with `{` → `ErrMalformed`.
3. Unmarshal with unknown fields tolerated; missing `fixes` → empty slice.
4. Validation (each failure → `ErrMalformed` with a short reason, surfaced as
   exit 4 by issue 16): `explanation` non-empty after sanitize, ≤ 4000 chars
   (truncate, don't fail, when longer); `fixes` > 5 → keep first 5; per fix —
   `command` non-empty, single line (reject fixes containing `\n` — drop that
   fix, not the whole response), ≤ 500 chars (drop if longer), `description`
   truncated at 200; `risk` normalized to {safe,caution,destructive}, unknown →
   `caution`; `confidence` normalized to {low,medium,high}, unknown → `low`.
   If ALL fixes were dropped but explanation is valid → valid zero-fix Result.
5. Sanitization: `sanitize.Strip` applied to every string field BEFORE
   validation (so length caps apply to clean text) — DESIGN §10.4.4.
6. Danger overlay (DESIGN §10.4.5): regex list covering at minimum
   `rm -rf /`, `rm -rf ~`, standalone `sudo` prefix, `curl … | sh` /
   `wget … | sh` (any sh/bash/zsh), `git push --force` (and `-f`),
   `chmod -R 777`, `dd of=/dev/`, `mkfs`, fork bomb `:(){ :|:& };:`.
   Any match forces `Risk = "destructive"`; overlay never lowers a risk;
   list is append-only (comment in file).
7. Package exposes `ErrMalformed` as a sentinel (`errors.Is`-able).

## Acceptance Criteria

- [ ] Corpus tests: clean JSON; fenced JSON (with and without `json` tag);
      chatty prose around JSON; leading BOM/whitespace; unknown extra fields;
      6 fixes → 5 kept; multi-line command dropped; 5000-char explanation
      truncated to 4000; empty object → ErrMalformed; plain prose → ErrMalformed.
- [ ] ANSI/OSC escapes inside every field are gone from `Result` (fixture with
      escapes in explanation, command, description).
- [ ] Danger table: each listed pattern flips risk to destructive; a
      model-claimed `destructive` with no pattern match stays destructive
      (never lowered); `safe`+`sudo rm` → destructive.
- [ ] Zero-fix valid response renders as valid Result (len 0).
- [ ] `errors.Is(err, ErrMalformed)` works for all failure paths.

## Validation

`go test ./internal/parse/...`; adversarial corpus reviewed in PR; fuzz test
(`go test -fuzz=FuzzParse -fuzztime=30s` locally, seed corpus committed) that
asserts: never panics, output strings never contain ESC bytes.

## Dependencies

06, 14 (schema/prompt pairing — the §10.4 schema constants referenced from the
prompt template must be single-sourced or cross-tested to avoid drift).

## Non-goals

Rendering (17); retry-on-malformed (v2 / U4); executing anything (INV-1).

## Design References

DESIGN.md §10.4, §9.3, §12.2 T4/T9, §12.3 INV-1/INV-2.
