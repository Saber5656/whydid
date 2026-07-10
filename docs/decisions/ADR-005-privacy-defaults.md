# ADR-005: Privacy Defaults — Standard Payload, Mandatory Redaction, Consent, Inspectability

Date: 2026-07-08
Status: Accepted (confirmed with product owner, 2026-07-08)

## Context

whydid sends terminal context to an external LLM API. That context routinely
contains secrets (tokens pasted into commands, keys echoed in error output),
personal paths, and hostnames. The payload scope options were "minimal"
(command + exit code + error output), "standard" (adds normalized cwd and
platform info), and "rich" (adds recent command history and git state).

## Decision

1. **Standard payload as the default and the v1 maximum.** The payload contains
   exactly: the failed command (redacted), exit code, captured error-output tail
   (redacted, only when available), cwd with `$HOME` abbreviated to `~`, shell
   name, OS name. **Command history is never sent in v1.** The rich tier is
   deferred to v2 as an explicit opt-in.
2. **Redaction is mandatory and not user-disableable.** A built-in pattern set
   (DESIGN.md §7) masks known credential formats and sensitive key=value pairs in
   BOTH persistence and payload paths. Users may add custom patterns; they cannot
   turn the built-ins off in v1.
3. **Redact at record time.** Command lines are redacted before they are written
   to the local store, so plaintext secrets from command lines never rest on disk.
   Captured pane output is never persisted at all.
4. **First-run consent.** The first `whydid` explain run shows: provider,
   endpoint host, exact field list to be sent, and the redaction notice; the user
   must confirm once (`privacy.consent_version` recorded). Non-interactive
   invocation without prior consent fails closed with instructions.
5. **Inspectability.** `whydid --show-payload` renders the exact payload
   (post-redaction) that would be sent and exits without any network call.
6. **No telemetry, ever.** whydid makes network connections only to the
   user-configured LLM endpoint. Release builds contain no analytics, update
   pings, or crash reporting.
7. **Keys are read, never written.** API keys come from environment variables or
   a user-configured `api_key_cmd` (e.g. a password-manager CLI). whydid never
   writes key material to config, state, logs, or error messages.

## Consequences

- Path/environment-dependent failures (wrong cwd, missing file) remain explainable
  thanks to cwd + platform fields; identity-adjacent data (history, git remotes,
  usernames) stays local.
- Redaction can occasionally mask non-secrets (false positives); the payload
  preview makes this visible, and custom allow-listing is deliberately absent in
  v1 (fail closed).
- A redacted command may make some fixes less precise (placeholders in the fix);
  this is the accepted cost of default-safe behavior.
- Consent + preview + no-telemetry are load-bearing for OSS trust and are encoded
  as acceptance criteria in issues 06, 19, 22.
