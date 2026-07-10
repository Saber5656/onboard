# Title

Chat backend: retrieval, prompt assembly, SSE streaming, budgets

## Summary

Implement `src/chat/` and mount `POST /api/chat` on the serve server per DESIGN
§10.3–10.6: request validation (token, body schema, size), deterministic BM25 retrieval
+ context assembly within the 12k-char budget, provider streaming via the adapters, the
SSE event protocol (`meta`/`delta`/`usage`/`done`/`error`), session token accounting
against `chat.maxTokensPerSession`, and the in-flight stream cap.

## Context

This is the serve-time LLM boundary: keys stay in-process (ADR-003), context comes only
from generated artifacts (token-optimization thesis), and every answer's cost is
reported to the UI (35).

## Scope

- `src/chat/context.ts` — `buildContext(bundle, index, req) → { blocks, contextRefs }` (§10.4 query side via 26's `queryIndex`; boost current tourId/stepId docs ×1.3; assemble ≤ 12,000 chars: steps verbatim, symbols as signature lines, excerpts fenced with `path:start–end` headers).
- `src/chat/prompt.ts` — system prompt verbatim §10.4 + `<context>`-delimited blocks + history (≤ 8) + question.
- `src/chat/handler.ts` — the HTTP handler: guards (33) already ran; validate `X-Onboard-Token` (401), body zod schema (§10.3 shapes/limits → 400), body ≤ 64 KB (413), chat enabled (409), in-flight ≤ 2 (429 `busy`), session budget (429 `budget_exceeded`); stream SSE events exactly §10.3; wire real `sessionTokensUsed` into `/api/session` (33).
- `src/chat/accounting.ts` — per-process counters (session totals, per-request usage from adapter `usage` items; estimator §8.5 fallback marked `estimated`).
- Integration tests with the mock adapter (24).

## Detailed Requirements

1. SSE mechanics: `Content-Type: text/event-stream`, `Cache-Control: no-store`,
   heartbeat comment every 15 s while waiting on the provider; events serialized as
   `event: <name>\ndata: <json>\n\n`; client abort (socket close) aborts the provider
   call via AbortSignal; `error` event then stream end on provider failure (502 only
   when failure precedes any SSE byte — after streaming starts, errors are in-band).
2. `meta` event first, carrying `contextRefs` (§10.3) in retrieval-rank order.
3. History validation: roles alternate ending with the new user question; violations →
   400 `invalid-history`. History is included in the prompt but **not** re-retrieved
   (context comes from the current question + position only — documented).
4. Budget accounting: session total updates from provider-reported usage; the request
   that crosses the cap completes, subsequent requests get 429 (§10.3 note in DESIGN:
   "exceeded → 429" interpreted as: check before dispatch; in-flight completion allowed).
   `usage` SSE event includes `sessionTotal`.
5. Question length 1..2000 chars enforced by schema; answer `max_tokens` =
   `chat.maxOutputTokensPerAnswer`.
6. No filesystem reads per request except the bundle loaded once at serve start
   (in-memory); no repo file access at chat time (token-optimization + T1 hygiene).
7. Keys: adapter created once at serve start via 24; absence already surfaced by
   `/api/session` (33) — handler double-checks and 409s with `no-api-key` reason.

## Acceptance Criteria

- [ ] Happy path (mock adapter scripted): events arrive in order meta → deltas → usage → done; `contextRefs` non-empty for a mini-express-app question ("how are users created" — top ref is a userService symbol/step per 26's test); assembled context ≤ 12,000 chars (captured prompt assert).
- [ ] Missing/wrong token → 401 before any SSE; bad Origin already 403 (33 regression); malformed body → 400; 3rd concurrent stream → 429 `busy`.
- [ ] Session budget: cap set to fit one answer → second request 429 `budget_exceeded`; `/api/session` reflects the spent total.
- [ ] Client disconnect mid-stream → provider AbortSignal fired within 100 ms (mock asserts).
- [ ] Provider error before first delta → 502 JSON; after first delta → in-band `error` event and clean stream end; no unhandled rejections.
- [ ] Prompt snapshot test: system text + context delimiters + history layout frozen; injected repo content stays inside `<context>` blocks (assert a `"ignore previous instructions"` string in an excerpt lands only inside delimiters — T6 hygiene).

## Validation

`pnpm --filter onboard-cli-placeholder test` (chat suite; offline). Manual keyed
smoke against a real provider is deferred to issue 41.

## Dependencies

24 (adapters), 26 (queryIndex), 33 (server mount + session wiring).

## Non-goals

Chat UI (35); multi-session persistence (in-memory per process); retrieval beyond BM25
(v2 embeddings); tool use of any kind (§11.7 — the model can only produce text).

## Design References

DESIGN §10.3–10.6, §8.5 (estimator), §11.2 T4/T6, §11.7, ADR-003.
