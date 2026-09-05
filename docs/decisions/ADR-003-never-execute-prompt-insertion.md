# ADR-003: whydid Never Executes Commands; Fixes Are Delivered by Prompt Insertion

Date: 2026-07-08
Status: Accepted (confirmed with product owner, 2026-07-08)

## Context

The fix-delivery options were: display only, copy to clipboard, prompt insertion,
or confirmed direct execution (thefuck's model). LLM-generated commands are
untrusted output; any path where the tool itself spawns them inherits command
injection, misparse, and prompt-injection-via-error-output risks (see
DESIGN.md §12 threats T4/T9).

## Decision

- **Invariant INV-1: the whydid binary never executes, spawns, or evals a user
  command or a model-proposed command.** There is no `--exec` flag in v1, and
  issues must not add one.
- The selected fix is delivered by **prompt insertion**, performed by the shell
  wrapper function (only the parent shell can mutate its own input state):
  - zsh: `print -z -- "$fix"` (pre-fills the next editing buffer),
  - bash: `history -s -- "$fix"` + a printed hint ("press ↑ to load the fix"),
    since bash offers no supported cross-process buffer prefill.
- Execution therefore always requires a human to press Enter with the command
  visible in their own prompt.
- **Invariant INV-2: what is displayed is what is inserted.** The binary prints
  the selected fix, byte-identical to the sanitized string that was rendered,
  as the sole stdout payload (DESIGN.md §9.2). The wrapper never transforms it.

## Consequences

- Eliminates the entire "tool executed something destructive" class of failures;
  the security review scope for fixes reduces to display integrity (ANSI
  sanitization, INV-2) and human review.
- Slightly weaker convenience than auto-execution; mitigated by one-keystroke
  selection and buffer pre-fill.
- bash UX is a documented one-step degradation (↑ then Enter).
- Danger tagging of proposed fixes (rm -rf, sudo, curl|sh, force-push, etc.) is
  advisory display metadata only — never a gate that implies safe-to-run for
  untagged commands (DESIGN.md §10.4).
