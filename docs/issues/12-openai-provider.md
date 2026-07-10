# Title

OpenAI-compatible provider backend

## Summary

Implement `internal/llm/openaicompat`: the chat-completions backend used for
OpenAI itself and every OpenAI-compatible endpoint (Ollama, self-hosted
gateways) via `base_url` override.

## Context

This is the default provider (config `llm.provider = "openai"`) and the bridge
to local models (ADR-002). Wire format per DESIGN §10.5.1; shared policy comes
from issue 11.

## Scope

- `internal/llm/openaicompat/provider.go`
- Registration in the issue-11 factory

## Detailed Requirements

1. Endpoint: `POST {base}/chat/completions` where `base` =
   `cfg.BaseURL` or default `https://api.openai.com/v1` (trailing-slash
   normalization: exactly one `/` between base and path).
2. Headers: `Authorization: Bearer <key>` (key from issue-11 resolution),
   `Content-Type: application/json`.
3. Request body:

   ```json
   {"model": cfg.Model,
    "messages": [{"role":"system","content": r.System},
                  {"role":"user","content": r.User}],
    "max_tokens": r.MaxTokens,
    "response_format": {"type":"json_object"}}
   ```

   `response_format` included only when `cfg.JSONMode` (default true; known
   unknown U7 documents the `json_mode=false` escape hatch).
4. Response handling: decode `choices[0].message.content` (string); missing
   choices/empty content → `E_PROVIDER` ("provider returned an empty response").
   Error responses: decode `{"error":{"message":...}}` when present and pass
   through `SafeProviderMessage`.
5. Implementer note (from ADR-002): verify the current OpenAI chat-completions
   request/response field names against official docs at implementation time;
   the shapes above are the design-time contract — deviations must be reported
   back to DESIGN.md, not silently patched.
6. No usage of any OpenAI SDK (raw net/http via issue-11 `doJSON`).

## Acceptance Criteria

- [ ] httptest fake asserting: exact path `/v1/chat/completions` under default
      base; Bearer header; body fields incl. `response_format` present/absent by
      `json_mode`.
- [ ] Happy path returns the fake's content string verbatim.
- [ ] Fake variants: 401 → `E_AUTH`; 429+Retry-After → one retry then `E_RATE`;
      500 twice → `E_PROVIDER`; malformed JSON body → `E_PROVIDER`; empty
      choices → `E_PROVIDER`.
- [ ] `base_url=http://127.0.0.1:<port>/v1` (loopback http) works — the Ollama
      shape (`/v1/chat/completions`) is covered by a dedicated test case.
- [ ] Provider error containing ANSI escapes is sanitized in the returned error.

## Validation

`go test ./internal/llm/openaicompat/...` against httptest doubles; no live API
calls (DESIGN §14).

## Dependencies

11.

## Non-goals

Anthropic backend (13); streaming; tool/function calling; token accounting.

## Design References

DESIGN.md §10.5.1, §10.6; ADR-002; research/01 §5.
