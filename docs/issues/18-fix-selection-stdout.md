# Title

Fix selection loop and the stdout contract (INV-2)

## Summary

Implement the interactive selection prompt on /dev/tty and the stdout emission
of the chosen fix — the binary half of the prompt-insertion chain whose shell
half lives in the init snippets (08/09).

## Context

INV-2 ("displayed fix == inserted fix, byte-identical") is enforced here: the
same `parse.Fix.Command` string object that the renderer displayed is the only
thing ever written to stdout (DESIGN §9.2). The selection loop reads from
/dev/tty because stdin/stdout are not usable inside `$( … )` capture.

## Scope

- `internal/render/select.go` — `Select(tty io.ReadWriter, res parse.Result,
  showPayload func(io.Writer)) (chosen *parse.Fix, err error)`
- `internal/cli/tty.go` — `OpenTTY() (io.ReadWriteCloser, error)` opening
  `/dev/tty` (injectable in tests)
- Emission helper `EmitFix(w io.Writer, fix parse.Fix)` writing
  `fix.Command + "\n"` and nothing else

## Detailed Requirements

1. Prompt text exactly: `Apply fix [1-N], (s)how payload, (q)uit: ` (N = fix
   count) written to the tty writer; read one line from the tty reader.
2. Input handling: digits 1..N → return that fix; `s`/`S` → call
   `showPayload(tty)` then re-prompt (does not count as an attempt); `q`/`Q`/
   empty-after-3-invalid → return nil (quit); invalid input → `invalid choice`
   + re-prompt, max 3 invalid attempts then quit (DESIGN §9.3).
3. Zero fixes → skip selection entirely (caller behavior, but `Select` must
   handle it by returning nil immediately).
4. `EmitFix` writes exactly `command\n` — no quoting, no color, no trailing
   spaces (DESIGN §9.2). It is the ONLY function in the codebase allowed to
   write non-empty explain-flow output to stdout (comment + issue-22 audit).
5. Ctrl-C / EOF on tty read → treat as quit (nil, no error).
6. `OpenTTY` failure is not handled here — the caller (16) routes to
   non-interactive mode; `Select` assumes a working tty pair.

## Acceptance Criteria

- [ ] Table tests driving `Select` with scripted tty input: `1` → first fix;
      `2\n` → second; `s` then `1` → payload callback invoked once, then first
      fix; `q` → nil; `x\nx\nx\n` → nil after 3 invalids; EOF → nil.
- [ ] Byte-identity test: for a fix command containing unusual-but-legal bytes
      (UTF-8 kanji, `$VAR`, quotes, backslashes), `EmitFix` output ==
      `[]byte(fix.Command + "\n")` and equals the string the renderer displayed
      (shared-struct assertion with issue 17's golden input).
- [ ] Nothing is written to the emission writer when selection returns nil.
- [ ] Prompt and errors go to the tty writer, never to the emission writer.

## Validation

`go test ./internal/render/...` with in-memory tty doubles; the PTY-level proof
that zsh `print -z` receives the bytes happens in issue 23.

## Dependencies

17.

## Non-goals

The shell wrapper functions (08/09 already ship them); arrow-key/TUI selection
(v1 is number keys only); executing anything (INV-1).

## Design References

DESIGN.md §9.2–9.3, §12.3 INV-2; ADR-003.
