# Title

LLM core: provider interface, HTTP policy, key resolution

## Summary

Implement `internal/llm`: the `Provider` interface and factory, the shared HTTP
client policy (timeouts, single retry, https-or-loopback rule), the typed error
taxonomy, and API-key resolution (env → api_key_cmd) with strict no-leak rules.

## Context

Both backends (12, 13) plug into this. The https rule mitigates T6; key handling
rules implement INV-5/T5 (DESIGN §11.4). Errors here surface as exit codes 2/3
via explain (16).

## Scope

- `internal/llm/llm.go` — `Request`, `Provider`, `New(cfg config.LLM)` factory
- `internal/llm/http.go` — shared `doJSON` helper (client construction, retry,
  status mapping)
- `internal/llm/errors.go` — `E_NO_KEY, E_AUTH, E_RATE, E_PROVIDER, E_NETWORK,
  E_TIMEOUT, E_INSECURE_ENDPOINT` as typed errors with user messages
- `internal/llm/key.go` — `ResolveKey(cfg config.LLM, provider string) (string, string, error)`
  returning (key, sourceDescription, error)

## Detailed Requirements

1. Interface exactly per DESIGN §10.1 (`Request{System, User, MaxTokens}`,
   `Explain(ctx, r) (string, error)`); `New` returns the backend named by
   `cfg.Provider` or a config error.
2. Key resolution order (DESIGN §10.1): `WHYDID_API_KEY` → env named by
   `cfg.APIKeyEnv`, defaulting per provider (`OPENAI_API_KEY` /
   `ANTHROPIC_API_KEY`) → `cfg.APIKeyCmd`. The command runs via
   `exec.CommandContext` **without a shell**, argv from the config array, 5 s
   timeout, stdin closed, env limited to `PATH` and `HOME` only; output =
   trimmed first line; empty output or failure → try nothing further → `E_NO_KEY`.
   `sourceDescription` examples: `env:WHYDID_API_KEY`, `env:OPENAI_API_KEY`,
   `api_key_cmd`, used by doctor (21) — never includes the key value.
3. Endpoint policy (`http.go`): parse the final URL; scheme `https` required
   unless hostname ∈ {`localhost`, `127.0.0.1`, `::1`} → `E_INSECURE_ENDPOINT`
   otherwise (DESIGN §10.6). No proxy handling beyond Go's defaults.
4. Retry policy: exactly one retry, only for connect errors, 429, and 5xx;
   honor a parseable `Retry-After` seconds header capped at the remaining
   context budget, else 1 s. Whole call bounded by
   `context.WithTimeout(cfg.TimeoutSeconds)` — set by the caller (16) but
   enforced defensively here too.
5. Status mapping: 401/403 → `E_AUTH`; 429 (post-retry) → `E_RATE`; other
   non-2xx → `E_PROVIDER`; transport errors → `E_NETWORK`; deadline →
   `E_TIMEOUT`.
6. Error hygiene (INV-5/T5): error strings may include the provider's
   error-message field (extracted by backends), truncated to 300 chars and
   passed through `sanitize.Strip`; they must never include the request body,
   URL query, or any header. Add a helper `SafeProviderMessage(raw string) string`.
7. `doJSON(ctx, method, url, headers, body any, out any) error` centralizes:
   marshaling, `Content-Type: application/json`, status mapping, retry,
   response size cap 1 MiB.

## Acceptance Criteria

- [ ] Factory returns the right backend or a named config error.
- [ ] Key resolution: table tests for each source, precedence, api_key_cmd
      timeout/failure, and the no-key terminal error; asserts the key value
      never appears in returned errors or sourceDescription.
- [ ] `http://example.com` → `E_INSECURE_ENDPOINT`; `http://127.0.0.1:11434` OK.
- [ ] Retry: httptest server failing once with 500 then 200 → success with
      exactly 2 requests; 429 with `Retry-After: 1` waits ~1 s.
- [ ] All taxonomy errors map from synthetic responses; each has a distinct,
      actionable one-line message.
- [ ] No logging anywhere in the package.

## Validation

`go test ./internal/llm/...` with `httptest.Server`; race detector on.

## Dependencies

04.

## Non-goals

Wire formats (12/13); prompt content (14); streaming; custom CA options (v2).

## Design References

DESIGN.md §10.1, §10.6, §11.4, §12.2 T5/T6, §12.3 INV-5.
