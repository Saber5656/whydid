# Title

`whydid config` subcommand

## Summary

Implement `whydid config get|set|list|path`: safe, validated read/write access
to the TOML config from the command line.

## Context

Users set `llm.model` and friends without editing TOML by hand
(DESIGN §11.1/§11.3). `set` must validate before writing so a typo cannot brick
the explain flow.

## Scope

- `internal/cli/configcmd.go` (subcommand wiring + argument parsing)
- Reuses `internal/config` (04) exclusively — no new config logic

## Detailed Requirements

1. Key addressing: dotted paths matching the TOML structure exactly
   (`llm.model`, `llm.provider`, `context.capture`, `ui.language`,
   `redaction.custom_patterns`, `privacy.consent_version`, …). Supported keys
   are enumerated in one table (key → type → settable) generated from the
   config struct via reflection or a hand-map with a completeness unit test
   against the struct fields.
2. `get <key>`: prints the value in TOML literal form to stdout, exit 0;
   unknown key → error exit 2. Arrays print as TOML arrays.
3. `set <key> <value>`: parses value per the key's type (string / int / bool /
   string-array as TOML array literal e.g. `'["a","b"]'`); applies to a copy;
   runs full validation (issue 04); on pass → `config.Save`; on fail → the
   ValidationError message, exit 2, file untouched.
4. `set privacy.consent_version` is refused with "managed by whydid" (exit 2) —
   consent state changes only via the consent flow (ADR-005.4).
5. `set llm.api_key_env` etc. is allowed, but any attempt to set a key that
   *looks like key material* (key name containing `api_key` with a value
   matching `redact` built-ins) prints a warning "whydid never stores API
   keys — set the environment variable instead" and refuses (exit 2).
6. `list`: prints the effective config as TOML to stdout with the same
   template used by Save (defaults merged in); values pass through
   `sanitize.Strip` defensively.
7. `path`: prints the resolved config file path (respects `WHYDID_CONFIG`).
8. All human diagnostics to stderr; machine output (the value / TOML / path) to
   stdout — this subcommand is scripting-friendly.

## Acceptance Criteria

- [ ] `config set llm.model x && config get llm.model` round-trips.
- [ ] `config set llm.provider bogus` → exit 2, file byte-identical to before.
- [ ] `config set llm.timeout_seconds 0` → exit 2 (range rule from 04).
- [ ] `config set redaction.custom_patterns '["(?i)corp-[0-9]+"]'` persists and
      `config get` echoes the array.
- [ ] Consent-version set refused; api-key-value set refused with the warning.
- [ ] `config path` honors `WHYDID_CONFIG`.
- [ ] Key-table completeness test fails if a new config field is added without
      updating the table.

## Validation

CLI-level tests running the built binary with temp `$HOME`/`WHYDID_CONFIG`;
unit tests for value parsing.

## Dependencies

04.

## Non-goals

Interactive setup wizard (v2); editing comments in place (DESIGN §11.3 allows
rewrite); managing keys/secrets (never).

## Design References

DESIGN.md §11.1, §11.3–11.4; ADR-005.4/7.
