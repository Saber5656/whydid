# Title

`whydid init bash`: vendored bash-preexec integration

## Summary

Add the bash variant of the init snippet: vendor bash-preexec (MIT) into the
embedded assets, detect an already-loaded copy, register whydid's
preexec/precmd functions, and provide the history-based fix insertion wrapper.

## Context

bash has no native preexec; the vendored rcaloras/bash-preexec (v0.6.0, MIT,
used by iTerm2/Ghostty) is the standard shim (research/02 §2). Coexistence with
user-loaded copies is the main risk (known unknown U2).

## Scope

- `internal/shellassets/vendor/bash-preexec.sh` — pinned v0.6.0 content, with a
  header comment recording upstream URL, version, license, and the upstream file
  SHA-256 (DESIGN §12.7)
- `internal/shellassets/whydid.bash`
- `internal/cli`: enable `whydid init bash`

## Detailed Requirements

1. Snippet guards mirror zsh: interactive (`[[ $- == *i* ]]`), `WHYDID_DISABLE`,
   binary presence; then `export WHYDID_SESSION_ID="bash-$$-<rand6>"`
   (rand6 from `$RANDOM` arithmetic, no subprocesses).
2. bash-preexec loading rule (research/02 §2): if
   `declare -p preexec_functions >/dev/null 2>&1` fails, emit/source the
   vendored `bash-preexec.sh` inline (the `init` output includes its full text);
   otherwise reuse the existing arrays. Never source a second copy.
3. `__whydid_preexec` stores `$1` and a start timestamp: use `$EPOCHSECONDS`
   when set, else `$(date +%s)` (bash 3.2 fallback; whole seconds accepted —
   DESIGN §5.3.3).
4. `__whydid_precmd` mirrors DESIGN §5.2.4 with `--shell=bash`; `--tty` uses a
   value cached once at init time via `tty` (research/02: one-time cost only).
5. Functions are appended to `preexec_functions` / `precmd_functions` arrays
   (idempotent: skip if already present).
6. Wrapper function per DESIGN §5.3.5: capture stdout; when non-empty,
   `history -s -- "$fix"` and print the ↑-hint to stderr.
7. Snippet must work on bash 3.2 (macOS /bin/bash) and bash 5 (Linux): no
   associative arrays, no `${var,,}`, no `EPOCHREALTIME` dependence.
8. `whydid init bash` output = header + vendored bash-preexec (conditional
   sourcing form) + whydid functions, single self-contained eval-able stream.

## Acceptance Criteria

- [ ] `bash -ic 'eval "$(whydid init bash)"; false; exit'` stores one record
      with `exit:1`, `shell:"bash"`.
- [ ] With a pre-sourced upstream bash-preexec, init does not double-load
      (exactly one record per command; `__bp` guard variables unique).
- [ ] Works on bash 3.2 syntax: `bash --posix`-independent, verified via
      `bash -n` and a macOS CI job step.
- [ ] Vendored file is byte-identical to upstream v0.6.0 (SHA-256 in header
      matches; a unit test recomputes the hash of the embedded body).
- [ ] Wrapper: a fake fix on stdout ends up retrievable via `history 1`
      containing the exact fix string.
- [ ] `WHYDID_DISABLE=1` → no-op.

## Validation

Same Go-driven real-shell tests as issue 08, run for bash on both CI OSes;
hash-pin unit test for the vendored asset.

## Dependencies

07, 08 (init subcommand plumbing and asset embedding created there).

## Non-goals

READLINE buffer prefill (unsupported cross-process; ADR-003 documents the ↑
degradation); fish; login-shell rc-file editing (user adds the eval line
themselves; docs in issue 27).

## Design References

DESIGN.md §5.1, §5.3–5.5, §12.7; research/02 §2–§4; ADR-003.
