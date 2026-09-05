# Review resolution record

- Repository: `Saber5656/whydid`
- Pull request: #1
- Parent head observed before this addendum: `2dfb042541f8689ac4a9a048075928fea4592424`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTNj_286P2UiO`

### Redact keys echoed in provider errors

- Finding: The existing review thread `PRRT_kwDOTNj_286P2UiO` identifies this contract gap.
- Normative resolution: Run every provider error body through the same secret redaction before storing, returning, or printing it; redact configured keys and recognizable bearer/API-key forms even when echoed by a provider.
- Focused verification before resolving this thread: Use a provider fixture that echoes the submitted key and assert error records, explain output, and logs contain only the redacted form.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2UiQ`

### Gate doctor endpoint dials on consent

- Finding: The existing review thread `PRRT_kwDOTNj_286P2UiQ` identifies this contract gap.
- Normative resolution: Make doctor perform only local configuration/readiness checks before consent; endpoint DNS/TCP/TLS dials require current privacy consent and must be skipped otherwise.
- Focused verification before resolving this thread: Run doctor on a clean config with a network-capture endpoint and assert no dial occurs before consent.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2UiS`

### Resolve the target from the latest command first

- Finding: The existing review thread `PRRT_kwDOTNj_286P2UiS` identifies this contract gap.
- Normative resolution: Model the latest command record as the primary explain target; only use the latest relevant failure when the latest command itself is not explainable, never select an older nonzero record over a newer successful command.
- Focused verification before resolving this thread: Record `false; true; whydid` and assert explain follows the documented latest-command state.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2UiV`

### Let --show-payload run before consent

- Finding: The existing review thread `PRRT_kwDOTNj_286P2UiV` identifies this contract gap.
- Normative resolution: Make `--show-payload` a local, redacted preview path that can run before consent and before any network call; sending remains consent-gated and the preview clearly states it is not transmission.
- Focused verification before resolving this thread: Run on a clean config in no-TTY mode and assert the exact redacted preview is printed without dialing or requiring consent.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2Uia`

### Expose all outbound fields in consent

- Finding: The existing review thread `PRRT_kwDOTNj_286P2Uia` identifies this contract gap.
- Normative resolution: List all eight outbound payload fields, including `duration_ms` and `error_output_source`, in the consent display and golden contract; field additions require updating this list.
- Focused verification before resolving this thread: Compare consent output with the serialized payload schema and assert exact field coverage.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2Uie`

### Abbreviate HOME only on a path boundary

- Finding: The existing review thread `PRRT_kwDOTNj_286P2Uie` identifies this contract gap.
- Normative resolution: Replace HOME only when cwd equals HOME or starts with HOME plus the platform separator; adjacent paths such as `/home/alice2` remain unshortened.
- Focused verification before resolving this thread: Run HOME, HOME-child, and HOME-prefix-sibling paths and assert only the first two use `~`.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2Uii`

### Scope the exec audit away from test harnesses

- Finding: The existing review thread `PRRT_kwDOTNj_286P2Uii` identifies this contract gap.
- Normative resolution: Define the AST audit's production roots and an explicit test/fixture exclusion or allowlist, so intentional test subprocesses do not violate the production execution allowlist.
- Focused verification before resolving this thread: Place allowed production and test harness exec calls in fixtures and assert the audit flags only out-of-policy production calls.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2Uil`

### Sanitize stored commands before doctor prints them

- Finding: The existing review thread `PRRT_kwDOTNj_286P2Uil` identifies this contract gap.
- Normative resolution: Apply ANSI/C0/OSC/CSI sanitization at persistence/load and again at doctor rendering; redaction and control removal are separate invariants.
- Focused verification before resolving this thread: Persist a command containing ESC/OSC/CSI and assert doctor output and JSON reports contain no terminal controls.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2Uip`

### Pass through stdout-producing subcommands

- Finding: The existing review thread `PRRT_kwDOTNj_286P2Uip` identifies this contract gap.
- Normative resolution: Have the hook bypass capture/insertion for stdout-producing `version`, `help`, `config get`, `init`, and equivalent subcommands, preserving their stdout and exit status.
- Focused verification before resolving this thread: Invoke each subcommand through the installed hook and assert stdout reaches the terminal without entering the shell buffer/history.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2Uir`

### Avoid creating config files from the hook path

- Finding: The existing review thread `PRRT_kwDOTNj_286P2Uir` identifies this contract gap.
- Normative resolution: Use a read-only/default-in-memory config load from the prompt hook; only explicit `init` or config commands may create or rewrite files.
- Focused verification before resolving this thread: Run a fresh shell prompt with no config and assert no file is created, then run explicit init and assert creation occurs.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNj_286P2Uiw`

### Enforce annotated release tags

- Finding: The existing review thread `PRRT_kwDOTNj_286P2Uiw` identifies this contract gap.
- Normative resolution: Before release, verify the tag object type is `tag` (annotated) and reject lightweight `commit` tags even when the name matches `v*`.
- Focused verification before resolving this thread: Run the release guard with annotated and lightweight v-tags and assert only annotated tags proceed.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.