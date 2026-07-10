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

- `src/llm/adapter.ts` — interface (§10.5 verbatim; `name: string`) + `resolveApiKey(provider, env)` + `createAdapter(config.llm)` + `class LlmProviderError extends Error { code: "llm-provider-error" | "llm-malformed-stream" | "llm-timeout"; status?: number }` (caught by narration → template fallback; by chat → 502/in-band error; it never maps to its own exit code — an uncaught escape is a bug and exits 1).
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
   max_tokens, messages, stream: true, stream_options: { include_usage: true } }` where
   `messages` = `[{role:"system", content: req.system}, ...req.messages]` (the
   interface's separate `system` is prepended); parse `choices[0].delta.content`
   deltas, final chunk `usage {prompt_tokens, completion_tokens}` → usage item;
   `data: [DONE]` terminates.
4. SSE robustness (both providers): comment lines and unknown event types are ignored;
   invalid JSON in a recognized `data:` payload → `LlmProviderError("llm-malformed-stream")`.
5. Key resolution (§4.4): `ONBOARD_LLM_API_KEY` else provider var; missing →
   `UsageError("llm-key-missing")` naming the expected variables (never echoing values).
6. Hardening (§11.7): `fetch(..., { redirect: "error", signal })`; connect timeout 10 s
   (time to response headers) and idle timeout 60 s (inter-chunk), both via
   AbortController — adapter-internal timeouts throw `LlmProviderError("llm-timeout")`;
   a **caller-supplied** abort ends the iterator quietly (normal return, no throw).
   Non-2xx → `LlmProviderError` with status; the surfaced/logged detail is the first
   200 chars of the body **after** passing the logger redaction filter (provider errors
   can echo prompt fragments — never log raw). Request init objects are never logged.
7. Defense in depth: `createAdapter` re-asserts `baseUrl` protocol ∈ {http, https}
   (normative validation lives in 03; this is the backstop) and rejects otherwise with
   `UsageError("config-baseurl-invalid")`.
8. Mock adapter: constructor takes `{ script: Array<delta|usage|error|hangMs> }`;
   `name: "mock"` (interface `name` is `string` per §10.5); deterministic; exported for
   other issues' tests. It ships inside the package (tiny) but is unreachable from any
   CLI path: the config `provider` enum has only the two real values (enforced by the
   config schema; issue 37 later adds an env-gated, `NODE_ENV=test`-only seam for e2e).

## Acceptance Criteria

- [ ] Contract tests replay both transcript fixtures through the stub server: assembled text and usage numbers match frozen expectations; caller abort mid-stream ends the iterator quietly within 100 ms (no throw); the openai-compat request body's first message is the prepended system message.
- [ ] Redirect attempt (stub replies 302) → `LlmProviderError`, no second request (assert stub hit-count 1).
- [ ] Timeouts: a stub that never sends headers → `llm-timeout` at ~10 s (fake timers); a stub that stalls mid-stream → `llm-timeout` at the idle limit; no unhandled rejections (process-level listener assert).
- [ ] Malformed stream: invalid JSON in a `data:` line → `llm-malformed-stream` (both providers); SSE comments and unknown events are ignored without error.
- [ ] Provider error body containing a planted key-like string is logged redacted (canary absent from captured logs).
- [ ] `resolveApiKey`: precedence and missing-key error message list both env var names; with `ONBOARD_LLM_API_KEY` set, provider-specific vars are ignored.
- [ ] `createAdapter` protocol backstop: `file://` baseUrl → `config-baseurl-invalid`, no request attempted.
- [ ] `openai-compat` against a baseUrl with trailing slash and without produce identical request paths (`…/chat/completions`).
- [ ] Import-level test: no module in `src/llm/` imports from `"openai"` or `"@anthropic-ai/sdk"` (parse import specifiers — a plain grep would false-positive on `openai-compat.ts`), and package.json has neither dependency (ADR-008 guard).

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
