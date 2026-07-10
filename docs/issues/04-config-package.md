# Title

Config package: TOML schema, defaults, validation, paths

## Summary

Implement `internal/config`: load/validate the TOML config, create a commented
default file on first use, resolve XDG paths, and expose typed accessors used by
every other package.

## Context

The config file is the only persistent user-editable surface (DESIGN §11.3).
Validation failure semantics feed exit code 2 (DESIGN §9.1). This issue also
owns path resolution for the state dir so store (05) and doctor (21) share it.

## Scope

- `internal/config/config.go` — types, `Load`, `Default`, `Save`
- `internal/config/paths.go` — config path + state dir resolution
- `internal/config/validate.go`
- Add runtime dependency `github.com/BurntSushi/toml` (allowed by ADR-004)

## Detailed Requirements

1. Struct mirrors DESIGN §11.3 exactly (sections `llm`, `context`, `redaction`,
   `ui`, `privacy`, top-level `schema_version`). Field names/keys must match the
   TOML keys in DESIGN verbatim.
2. `ConfigPath()`: `$WHYDID_CONFIG` if set, else
   `${XDG_CONFIG_HOME:-$HOME/.config}/whydid/config.toml`.
   `StateDir()`: `$WHYDID_STATE_DIR` if set, else
   `${XDG_STATE_HOME:-$HOME/.local/state}/whydid`. Both pure functions of env.
3. `Load()` behavior: file missing → write `Default()` serialized **with the
   comment header and per-key comments shown in DESIGN §11.3** (keep a raw
   template string constant; do not rely on TOML-marshal comments), file mode
   0600, parent dir 0700, then return defaults. File present → parse, apply
   defaults for absent keys, validate.
4. Validation rules (fail → typed `ValidationError{Key, Msg}`):
   `llm.provider ∈ {openai, anthropic}`; `llm.timeout_seconds ∈ [1,300]`;
   `llm.max_tokens ∈ [64,8192]`; `context.error_output_lines ∈ [1,200]`;
   `context.capture ∈ {auto, off}`; `ui.color ∈ {auto, always, never}`;
   each `redaction.custom_patterns` entry compiles as RE2;
   `llm.api_key_cmd` if non-empty must have a non-empty argv[0];
   `llm.base_url` if non-empty must parse as URL with scheme http/https.
   Unknown keys: collect and expose as `Warnings []string` (loader does not
   print; callers decide — DESIGN §11.3).
5. `model` is NOT validated here (required only at explain time — DESIGN §11.3);
   expose helper `RequireModel() error`.
6. `Save(cfg)` rewrites the whole file from the template with current values
   (comment-preservation not required, per DESIGN §11.3), temp-file + rename,
   0600.
7. No I/O besides the config file; no logging.

## Acceptance Criteria

- [ ] First `Load()` on a clean HOME creates the commented default file with
      0600/0700 modes and returns defaults with zero warnings.
- [ ] Every validation rule above has a passing and a failing test case.
- [ ] Env overrides (`WHYDID_CONFIG`, `WHYDID_STATE_DIR`) are honored.
- [ ] Round-trip: `Save(Load())` is idempotent (stable output bytes).
- [ ] Unknown key `foo = 1` yields a warning, not an error.

## Validation

`go test ./internal/config/...` with a temp `$HOME`; golden file for the default
config template; `make lint` clean.

## Dependencies

01.

## Non-goals

`whydid config` CLI (issue 20); consent flow logic (19) — this package only
stores `privacy.consent_version`; key resolution (11).

## Design References

DESIGN.md §11.2–11.4, §9.1; ADR-004 (dependency policy), ADR-005.
