# Title

Anthropic Messages provider backend

## Summary

Implement `internal/llm/anthropic`: the Anthropic Messages API backend
(`llm.provider = "anthropic"`), including refusal-stop handling.

## Context

Second of the two v1 backends (ADR-002). Wire format per DESIGN §10.5.2; the
Messages API differs structurally from chat-completions (system as a top-level
field, content-block array in responses).

## Scope

- `internal/llm/anthropic/provider.go`
- Registration in the issue-11 factory

## Detailed Requirements

1. Endpoint: `POST {base}/v1/messages`, `base` = `cfg.BaseURL` or
   `https://api.anthropic.com`.
2. Headers: `x-api-key: <key>`, `anthropic-version: 2023-06-01`,
   `Content-Type: application/json`. (No `Authorization` header.)
3. Request body:

   ```json
   {"model": cfg.Model,
    "max_tokens": r.MaxTokens,
    "system": r.System,
    "messages": [{"role":"user","content": r.User}]}
   ```

4. Response handling: concatenate the `text` fields of all `content[]` blocks
   with `type == "text"`, in order. Empty result → `E_PROVIDER`.
5. If the response's `stop_reason` field equals `"refusal"`, return
   `E_PROVIDER` with the fixed message "the provider declined to answer this
   request" — do not retry (DESIGN §10.5.2). Note: `content` may be empty in
   this case; check `stop_reason` before the empty-content check.
6. API error responses: decode `{"error":{"message":...}}` shape when present →
   `SafeProviderMessage`.
7. Implementer note (ADR-002): verify current header names, version string, and
   response fields against Anthropic's official API reference at implementation
   time; report doc deviations back into DESIGN.md. Model IDs remain pure user
   config — never embed a default model.
8. No Anthropic SDK usage (raw net/http via issue-11 `doJSON`).

## Acceptance Criteria

- [ ] httptest fake asserting exact path `/v1/messages`, the three headers, and
      body shape (system top-level, single user message).
- [ ] Multi-block response (`[{type:"text",text:"A"},{type:"text",text:"B"}]`)
      returns `"AB"`; non-text blocks are skipped.
- [ ] `stop_reason:"refusal"` → `E_PROVIDER` with the fixed message; no retry
      request observed by the fake.
- [ ] 401 → `E_AUTH`; 529-style overload (5xx) → single retry then `E_PROVIDER`.
- [ ] Key never appears in any error string (fuzz the fake to echo the key in
      its error body: `SafeProviderMessage` output is truncated/sanitized but
      may contain it only if the *provider* echoed it — assert our code itself
      never adds it; and assert truncation at 300 chars).

## Validation

`go test ./internal/llm/anthropic/...` with httptest doubles only.

## Dependencies

11.

## Non-goals

Thinking/streaming/tool-use features; prompt caching; model selection logic.

## Design References

DESIGN.md §10.5.2, §10.6; ADR-002.
