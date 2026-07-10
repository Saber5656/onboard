# Title

LLM provider adapters (fetch/SSE) with key resolution and test mock

## Summary

Implement `src/llm/` per DESIGN §10.5 and ADR-008: the `LlmAdapter` interface, the
`anthropic` and `openai-compat` fetch-based streaming adapters with manual SSE parsing,
env-only key resolution, timeout/redirect hardening, and an in-process `mock` adapter
for tests (used by narration 25, chat 34, and e2e).

## Context

The only network code in the product. No SDKs (ADR-008); exact wire shapes are specified
so a lower-capability agent can implement from recorded transcripts without live keys.

## Scope

- `src/llm/adapter.ts` — interface (§10.5 verbatim) + `resolveApiKey(provider, env)` + `createAdapter(config.llm)`.
- `src/llm/anthropic.ts`, `src/llm/openai-compat.ts` — implementations.
- `src/llm/sse.ts` — line-based SSE parser (`data:` events, `[DONE]` handling, multi-line data).
- `src/llm/mock.ts` — deterministic mock (scripted deltas + usage; injectable failures/latency).
- `test/fixtures/llm-transcripts/` — recorded SSE bodies (text files) for both providers.
- Contract tests against a local `node:http` stub server replaying transcripts.

## Detailed Requirements

1. Interface exactly §10.5 (`streamComplete` yielding `delta`/`usage` items;
   AbortSignal respected — abort closes the connection and the iterator returns).
2. `anthropic`: POST `https://api.anthropic.com/v1/messages`; headers `x-api-key`,
   `anthropic-version: 2023-06-01`, `content-type: application/json`; body
   `{ model, max_tokens, temperature, system, messages, stream: true }`; parse events:
   `message_start` (capture `usage.input_tokens`), `content_block_delta`
   (`delta.text` → delta), `message_delta` (`usage.output_tokens`), `message_stop`.
   Emit one final `usage` item combining captured counts.
3. `openai-compat`: POST `{baseUrl}/chat/completions` (baseUrl from config, trailing
   slash normalized); header `Authorization: Bearer <key>`; body `{ model, temperature,
   max_tokens, messages: [{role:"system"|"user"|"assistant"}], stream: true,
   stream_options: { include_usage: true } }`; parse `choices[0].delta.content` deltas,
   final chunk `usage {prompt_tokens, completion_tokens}` → usage item; `data: [DONE]`
   terminates.
4. Key resolution (§4.4): `ONBOARD_LLM_API_KEY` else provider var; missing →
   `UsageError("llm-key-missing")` naming the expected variables (never echoing values).
5. Hardening (§11.7): `fetch(..., { redirect: "error", signal })`; connect timeout 10 s
   and idle timeout 60 s implemented via AbortController + inter-chunk timer; non-2xx →
   `LlmProviderError` with status + first 200 chars of body (body may contain provider
   error JSON — safe to log; keys never appear in requests' logs: only header names are
   loggable, enforced by never logging request init objects).
6. HTTP(S) only: `baseUrl` must parse as http/https URL (already zod-side in 03 — add
   the protocol check there if missing; assert here defensively too).
7. Mock adapter: constructor takes `{ script: Array<delta|usage|error|hangMs> }`;
   deterministic; exported for other issues' tests. It ships inside the package (it is
   tiny) but is unreachable from any CLI path: the config `provider` enum has only the
   two real values (enforced by the config schema; issue 37 later adds an env-gated,
   `NODE_ENV=test`-only seam for e2e).

## Acceptance Criteria

- [ ] Contract tests replay both transcript fixtures through the stub server: assembled text and usage numbers match frozen expectations; abort mid-stream closes within 100 ms.
- [ ] Redirect attempt (stub replies 302) → `LlmProviderError`, no second request (assert stub hit-count 1).
- [ ] Idle hang > timeout (mock server stalls) → error, iterator ends; no unhandled rejection (process-level listener assert).
- [ ] `resolveApiKey`: precedence and missing-key error message list both env var names; with `ONBOARD_LLM_API_KEY` set, provider-specific vars are ignored.
- [ ] `openai-compat` against a baseUrl with trailing slash and without produce identical request paths (`…/chat/completions`).
- [ ] grep-style test: `src/llm/` contains no import of `@anthropic-ai/sdk` or `openai` (ADR-008 guard).

## Validation

`pnpm --filter onboard-cli-placeholder test` (llm suite; fully offline — stub server on
127.0.0.1 ephemeral port).

## Dependencies

03 (llm config shape), 04 (error taxonomy).

## Non-goals

Narration/chat logic (25, 34); provider tool-use; retries beyond §10.5 policy (narration
retry lives in 25); token counting beyond provider-reported + §8.5 estimator.

## Design References

DESIGN §10.5, §11.7, §4.4, ADR-008.
