# ADR-008: No LLM provider SDKs — hand-rolled fetch adapters

Status: Accepted (2026-07-10)

## Context

The optional LLM layer (narration + chat) needs Anthropic and OpenAI-compatible
endpoints. Official SDKs (`@anthropic-ai/sdk`, `openai`) exist and are well-maintained,
but onboard's usage is a single streaming-completion call shape per provider.

## Decision

Implement two small adapters over `fetch` + manual SSE parsing (`anthropic`,
`openai-compat`), with request/response shapes specified exactly in DESIGN §10.5.
No provider SDK dependencies.

## Consequences

- Runtime dependency budget stays at 7 packages (DESIGN §11.8) — a meaningful
  supply-chain and install-weight win for a tool whose default mode needs no network at all.
- `openai-compat` + configurable `baseUrl` covers OpenAI, Ollama, LM Studio, OpenRouter,
  vLLM without extra code.
- We own SSE parsing and API-drift risk. Bounded: the two wire formats are stable and
  contract-tested against recorded transcripts (DESIGN §14.2); `redirect: "error"` and
  explicit timeouts are set on all requests (DESIGN §11.7).
- If provider APIs gain must-have complexity (tool use, etc. — not v1 features), revisit.

## Alternatives considered

- **Official SDKs**: lower maintenance on paper, but two heavyweight dependency trees for
  one endpoint each, plus SDK majors moving faster than the wire format. Rejected for v1.
