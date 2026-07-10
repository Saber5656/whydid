# Title

Renderer: explanation/fixes UI on stderr

## Summary

Implement `internal/render`: the stderr presentation of a parsed result — the
"why it failed" section, the context-used line, the numbered fix list with
danger tags — with color handling and width-aware wrapping.

## Context

All human output goes to stderr because the wrapper function captures stdout
(DESIGN §9.2/§9.3). Strings arriving here are already sanitized (issue 15);
the renderer must not introduce new interpretation of them.

## Scope

- `internal/render/render.go` — `Explanation(w io.Writer, res parse.Result,
  ctxLine string, opts Options)`
- `internal/render/color.go` — `Options{Color: auto|always|never, Width int}`,
  NO_COLOR handling, TTY detection helper (injectable for tests)
- `internal/render/wrap.go` — greedy word wrap (unicode-aware by rune count;
  East-Asian width precision is NOT required in v1)

## Detailed Requirements

1. Layout (match DESIGN §2.2 example):

   ```
   ── why it failed ────…(rule to width)
   <explanation, wrapped>
     context used: <ctxLine>
   ── fixes ────────────…
     1. <command>
        <description, dimmed>
     2. ⚠ <command>            [destructive]
   ```

   Zero fixes → omit the fixes section, print `no fixes proposed` dimmed.
2. `ctxLine` is composed by the caller (issue 16) from capture Result — the
   renderer prints it verbatim.
3. Color rules (DESIGN §9.3): `always` → on; `never` → off; `auto` → on iff
   `w` is a terminal AND `NO_COLOR` unset. Colors: section rules dim; command
   bold; description dim; `⚠`+`[destructive]` red+bold; `[caution]` yellow.
   `risk == safe` renders no tag.
4. Width: `Options.Width` (caller passes terminal width or 0); 0 or < 40 →
   no wrapping, plain rules of fixed 40 chars (DESIGN §13 E11).
5. The renderer never modifies command strings (INV-2 is about bytes; display
   uses the exact `parse.Fix.Command` — wrapping may only occur BETWEEN the
   number prefix and the command by placing the command on its own line, never
   inside the command; long commands overflow rather than wrap).
6. Pure functions over `io.Writer`; no direct os.Stderr references (caller
   wires it); no global state.

## Acceptance Criteria

- [ ] Golden tests (color=never) for: 2-fix example of DESIGN §2.2;
      destructive+caution tags; zero fixes; narrow width (<40) fallback;
      long-command overflow (command line un-wrapped).
- [ ] Golden test with color=always asserting exact SGR byte placement.
- [ ] `NO_COLOR=1` + auto + TTY → no SGR bytes.
- [ ] A command string containing `%s`/`%d` renders literally (no printf
      injection — use Fprint-style writes, covered by test).
- [ ] Renderer output for a Result equals renderer output for the same Result
      rendered twice (determinism).

## Validation

`go test ./internal/render/...` golden files; visual sample in PR description
(both shells' typical widths 80/120).

## Dependencies

06 (sanitize contract upstream), 10 (context line semantics), 15 (input types).

## Non-goals

Selection input loop and stdout emission (18); spinner (16); markdown rendering
of explanations (explanations are plain text by prompt contract).

## Design References

DESIGN.md §2.2, §9.2–9.3, §13 E11; ADR-003 (display integrity).
