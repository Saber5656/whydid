# Research: Shell Integration Techniques (zsh / bash / tmux)

Date: 2026-07-08
Status: Informative (feeds DESIGN.md §5, §6, §9 and issues 08, 09, 10, 18)

## Purpose

Verify the concrete mechanisms whydid's shell layer will rely on: pre/post command
hooks in zsh and bash, safe transport of command text, buffer insertion for the
"prompt insertion" UX, and tmux output capture.

## 1. zsh hooks (native)

zsh provides first-class hook arrays:

- `preexec` functions run just after a command line is read and about to execute;
  the typed command line is passed as `$1`.
- `precmd` functions run before each prompt; `$?` at that point is the exit status
  of the previous pipeline (must be captured as the very first statement).
- Register via `add-zsh-hook preexec <fn>` / `add-zsh-hook precmd <fn>`
  (autoload from `zsh/zle`), which composes with other plugins instead of
  overwriting `preexec()`/`precmd()` definitions.
- `$EPOCHREALTIME` (from `zsh/datetime` module) gives sub-second timestamps without
  spawning `date`.

Maturity: this is the same mechanism used by oh-my-zsh, powerlevel10k, Atuin, etc.
No known blockers.

## 2. bash hooks (via bash-preexec)

bash has no native preexec. The standard solution is **rcaloras/bash-preexec**:

- Implements `preexec_functions` / `precmd_functions` arrays on top of bash's
  `DEBUG` trap + `PROMPT_COMMAND`.
- MIT licensed; latest release v0.6.0 (2025-08-03); requires Bash 3.1+ (covers
  macOS's ancient /bin/bash 3.2 and all modern Linux bash 4/5).
- Used in production by iTerm2, Ghostty, and Bashhub — battle-tested.
- Constraints documented upstream:
  - It must be sourced **last** in the user's bash profile.
  - Anything that later overwrites the `DEBUG` trap or `PROMPT_COMMAND` wholesale
    will break it (it preserves pre-existing traps at install time, see PR #50).
  - Subshell support is off by default (`__bp_enable_subshells`) due to
    functrace/DEBUG-trap bugs — whydid does not need subshell events.

**Decision input:** vendor `bash-preexec.sh` (MIT, with attribution) inside the
whydid binary's embedded shell assets, but only install our functions if the arrays
are not already provided by a user-loaded copy (detect via
`declare -p preexec_functions`). This avoids double-loading conflicts with iTerm2
and Atuin users. `$EPOCHSECONDS` exists in bash 5; fall back to `$SECONDS`-based or
`date +%s` timestamps on bash 3.2/4.x.

## 3. Transporting the command text to the binary

The recorded command line is hostile input (may contain quotes, newlines, escape
bytes, secrets). Two constraints drive the transport design:

- **argv is world-readable** while the process runs (`/proc/<pid>/cmdline` on Linux
  is readable by other users; `ps` shows args on macOS). Command text and any
  captured output must therefore NOT be passed as CLI arguments.
- **Environment blocks** of other users' processes are not readable on Linux
  (`/proc/<pid>/environ` is 0400) and not shown cross-user by macOS `ps` for other
  users, but env size limits and quoting make env transport clumsy for arbitrary
  bytes.

**Decision input:** hooks pipe a single JSON object to `whydid hook record` via
**stdin**. JSON encoding is done by the binary's caller-side helper being trivial:
zsh/bash cannot safely JSON-encode arbitrary strings, so the hook instead passes
the command via stdin **raw** with a length-prefixed/heredoc-free framing and passes
only trivially-safe scalar fields (exit code, durations, cwd via `$PWD`) as
`--flag=value` arguments... — resolved design (see DESIGN.md §5.4): the hook writes
`exit_code`, timestamps, `cwd`, `shell`, `tty`, `tmux_pane` as argv flags (these are
low-risk, numeric or path-valued) and streams the raw command bytes on stdin. The
binary performs JSON encoding, redaction, truncation, and safe persistence.

## 4. Prompt insertion (the "fix lands in your input line" UX)

- **zsh:** `print -z -- "$cmd"` pushes text onto the ZLE editing buffer stack; the
  text appears as the pre-filled next input line. Standard, stable, used widely.
- **bash:** there is no supported way for a child process to pre-fill the parent's
  readline buffer. `READLINE_LINE` only works inside `bind -x` handlers. The
  portable degradation is `history -s -- "$cmd"`, which appends the fix to the
  in-memory history so the user presses ↑ then Enter. This is the same approach
  used by several completion/suggestion tools.
- In both shells the insertion is performed by a **shell function wrapper**
  (`whydid()` function defined by `whydid init`) because only the parent shell can
  mutate its own buffer/history. The binary communicates the chosen fix over
  stdout; everything interactive happens on /dev/tty (see DESIGN.md §9.2 stdout
  contract).

## 5. tmux output capture

- `tmux capture-pane -p -S -<N>` prints the last N lines of the current pane's
  scrollback to stdout ( `-p` = to stdout, `-S` = start line, negative = into
  history). `-J` joins wrapped lines. `$TMUX` env var presence identifies a tmux
  session; `$TMUX_PANE` identifies the pane to target explicitly.
- Capture is **on demand** at `whydid` invocation time — nothing is recorded
  continuously and nothing is written to disk. The captured text contains rendered
  screen content (prompts, command echo, output) and must be (a) segmented to the
  region belonging to the failed command, and (b) run through redaction before use.
- Segmentation heuristic that does not require prompt markers: search the captured
  text bottom-up for the last occurrence of the recorded command string (or its
  first line for multi-line commands); the error output is the text between that
  line and the capture end minus the current prompt line(s). If the command string
  cannot be found (redraws, clears, TUI apps), fall back to metadata-only payload
  and say so in the UI.

Limitations to document for users: pane must not have been cleared; full-screen TUI
apps (vim, less) leave no useful scrollback; output may be visually wrapped
(mitigated with `-J`).

## 6. Alternatives considered and rejected for v1 output capture

| Technique | Why rejected for v1 |
|---|---|
| `exec 2> >(tee ...)` stderr teeing in the hook | Turns stderr into a pipe: programs detect non-TTY and change behavior (colors off, different buffering); interleaving breaks; fragile with interactive programs |
| `script`-based whole-session recording (thefuck instant mode) | Continuous plaintext recording of everything including secrets; log management burden; prompt-marker fragility (see 01-prior-art.md) |
| Re-running the failed command | Unsafe for side-effectful commands; unacceptable per ADR-001 |
| Terminal-emulator-specific APIs (iTerm2, kitty, WezTerm) | Per-emulator work; niche coverage; deferred to v2 as additional OutputProviders |

## Sources

- zsh hook functions: zsh manual (zshmisc "SPECIAL FUNCTIONS", zshcontrib
  `add-zsh-hook`)
- bash-preexec: https://github.com/rcaloras/bash-preexec (MIT license, v0.6.0
  2025-08-03, Bash 3.1+, used by iTerm2/Ghostty/Bashhub; subshell caveats;
  "must be last imported"; DEBUG-trap preservation PR #50)
- tmux capture-pane: tmux(1) manual
- thefuck instant mode as prior art for `script`-based capture: see
  01-prior-art.md sources
