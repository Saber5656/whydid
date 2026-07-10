# Title

`whydid init zsh`: hooks, session id, wrapper function

## Summary

Implement the `whydid init <shell>` subcommand plumbing and the zsh snippet:
preexec/precmd hooks recording via `whydid hook record`, per-shell session id,
and the `whydid()` wrapper function that performs prompt insertion.

## Context

DESIGN §5.1–5.2 specifies the snippet normatively; research doc 02 §1/§4 records
the underlying zsh mechanisms. The wrapper function is half of the INV-2 fix
delivery chain (stdout contract lands in issue 18, but the wrapper's capture
behavior is fixed here).

## Scope

- `internal/shellassets/whydid.zsh` (embedded via `go:embed`)
- `internal/cli` `init` subcommand: `whydid init zsh` prints the asset to
  stdout; `whydid init bash` returns "not implemented" exit 5 until issue 09;
  unknown shell → usage + exit 2
- `internal/shellassets/assets.go` — embed + accessor

## Detailed Requirements

1. Snippet content must implement DESIGN §5.2 items 1–6 exactly, including:
   - top guards: `[[ -o interactive ]] || return 0`,
     `[[ -n "$WHYDID_DISABLE" ]] && return 0`,
     `command -v whydid >/dev/null 2>&1 || return 0`;
   - unconditional `export WHYDID_SESSION_ID="zsh-$$-<rand6>"` where rand6 is
     derived without spawning processes (e.g. from `$RANDOM$RANDOM` mapped to
     base36, exactly 6 chars) — DESIGN §5.1 nested-shell semantics;
   - `zmodload zsh/datetime`, `autoload -Uz add-zsh-hook`;
   - `__whydid_preexec` / `__whydid_precmd` exactly as §5.2.3–4 (first statement
     of precmd captures `$?`); re-registration guard `_WHYDID_HOOKED`;
   - wrapper function per §5.2.6 (`print -z -- "$fix"`, only when non-empty;
     propagate binary exit code).
2. Every command inside hooks uses `command whydid …` (bypasses the wrapper and
   user aliases).
3. The snippet must be pure zsh (no external processes besides the whydid
   binary) and must not change shell options, traps, or `$?` observable by the
   user's own precmds (restore semantics: hooks return 0).
4. `init` prints the snippet with a leading comment header naming the generating
   version (`# whydid init zsh (vX.Y.Z)`).
5. Add a `--print` no-op flag alias for forward compat (accepted, ignored).

## Acceptance Criteria

- [ ] `zsh -ic 'eval "$(whydid init zsh)"; false; exit'` leaves exactly one
      record with `exit:1`, `shell:"zsh"` in the session file (test via
      `WHYDID_STATE_DIR` temp dir).
- [ ] Sourcing the snippet twice registers hooks once (`_WHYDID_HOOKED` guard;
      record count for one command is 1).
- [ ] With `WHYDID_DISABLE=1`, sourcing is a no-op (no session id export, no
      records).
- [ ] `whydid` typed at the prompt produces no self-record (skip-self is binary
      side, but assert end-to-end here).
- [ ] Hook survives `whydid` binary being removed from PATH after init (silent
      no-op, prompt unaffected).
- [ ] Snippet passes `zsh -n` (syntax check) in CI.

## Validation

Go test compiling the binary and driving real `zsh -ic` scripts against a temp
state dir (predecessor of the fuller PTY harness in issue 23); assert store
contents with `internal/store` readers. CI has zsh on both matrix OSes
(issue 02).

## Dependencies

07.

## Non-goals

bash (09); the interactive selection/stdout contract internals (18); doctor's
rc-file checks (21).

## Design References

DESIGN.md §5.1–5.2, §5.4, §12.2 T8; research/02 §1, §4; ADR-003 (wrapper is the
insertion mechanism).
