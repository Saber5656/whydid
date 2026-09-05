# ADR-002: Cloud LLM with BYOK and a Two-Backend Provider Abstraction

Date: 2026-07-08
Status: Accepted (confirmed with product owner, 2026-07-08)

## Context

The explanation engine options were: cloud LLM (BYOK), local-LLM-only, a rule
engine, or a rules+LLM hybrid. A rule engine reproduces thefuck's maintenance
treadmill and caps explanation quality; local-only excludes most users at install
time. See docs/research/01-prior-art.md §3, §5.

## Decision

- v1 uses a **cloud LLM with user-supplied credentials (BYOK)**. whydid ships no
  keys, no proxy service, and no server-side component.
- Providers are implemented behind one small Go interface (`llm.Provider`) with
  exactly **two backends in v1**:
  1. `openai` — OpenAI **chat completions** wire format with a configurable
     `base_url`. Because Ollama and most self-hosted gateways expose the same
     `POST {base_url}/chat/completions` surface, this single backend also covers
     local/self-hosted models (e.g. `base_url = "http://localhost:11434/v1"`).
  2. `anthropic` — Anthropic Messages API (`POST /v1/messages`, `x-api-key`,
     `anthropic-version: 2023-06-01`).
- Both backends are implemented with Go's standard `net/http` and `encoding/json`.
  **No provider SDK dependencies** — the request surface whydid needs (one
  non-streaming chat call with a JSON-object response) is small, and zero SDKs
  keeps the supply chain minimal (see DESIGN.md §12.7).
- `model` is required user configuration. whydid never hardcodes model IDs in
  product logic; docs and the first-run flow may show examples, which implementers
  must verify against provider documentation at implementation time.
- No offline mode in v1: without network/key, whydid prints a clear error and the
  recorded metadata so the user can self-diagnose.

## Consequences

- Highest explanation quality per engineering hour; no rule corpus to maintain.
- Local/private operation is available on day one via the OpenAI-compatible
  backend pointed at Ollama — satisfying privacy-sensitive users without a
  separate engine.
- Sending command context to an external API becomes the product's central privacy
  risk; this is mitigated by mandatory redaction, a minimal default payload,
  first-run consent, and `--show-payload` (ADR-005, DESIGN.md §12).
- A rules-based offline fallback remains a v2 candidate (ISSUE_PLAN "Deferred v2").
