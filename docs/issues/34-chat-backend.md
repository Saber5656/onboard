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
- `src/chat/handler.ts` — the HTTP handler: guards (33) already ran; validate `Content-Type` is `application/json` (optionally with charset) → else 415; `X-Onboard-Token` (401); body via 33's `readLimitedJsonBody` (413) then zod schema (§10.3 shapes/limits → 400); chat enabled (409); 33's `StreamLimiter` (429 `busy`); session budget (429 `budget_exceeded`); stream SSE events exactly §10.3; wire real `sessionTokensUsed` into `/api/session` (33).
- `src/chat/accounting.ts` — per-process counters (session totals, per-request usage from adapter `usage` items; estimator §8.5 fallback marked `estimated`).
- Integration tests with the mock adapter (24).

## Detailed Requirements

1. SSE mechanics: `Content-Type: text/event-stream`, `Cache-Control: no-store`,
   heartbeat comment every 15 s while waiting on the provider; events serialized as
   `event: <name>\ndata: <json>\n\n`; client abort (socket close) aborts the provider
   call via AbortSignal. Error boundary (§10.3): provider failure **before any SSE
   byte is written** → 502 JSON; after any SSE byte (including after `meta`) → in-band
   `error` event, then clean stream end.
2. `meta` event first, carrying `contextRefs` (§10.3) as `{ type, id }` objects in
   retrieval-rank order (types are exactly the four §10.4 doc types).
3. Context boost mapping: `boostDocIds` = `step:<stepId>` plus `x:<excerptId>` of that
   step when `stepId` present; when only `tourId` is present, all `step:` ids of that
   tour. No other docs are boosted.
4. History validation (`history` = prior turns, excluding the new question): empty, or
   starts with `user`, strictly alternates, and ends with `assistant`; violations →
   400 `invalid-history`. The prompt appends the current question as the final `user`
   message. History is **not** re-retrieved (context comes from the current question +
   position only — documented).
5. Budget accounting: session total updates from provider-reported usage (the §8.5
   estimator is used **internally only** when a provider omits usage — the `usage` SSE
   event carries exactly the three §10.3 numeric fields, no `estimated` marker); the
   check runs before dispatch, in-flight completion allowed; subsequent requests 429.
6. Question length 1..2000 chars enforced by schema; answer `max_tokens` =
   `chat.maxOutputTokensPerAnswer`.
7. No filesystem reads per request except the bundle loaded once at serve start
   (in-memory); no repo file access at chat time (token-optimization + T1 hygiene).
8. Keys: adapter created once at serve start via 24; absence already surfaced by
   `/api/session` (33) — handler double-checks and 409s with `no-api-key` reason.

## Acceptance Criteria

- [ ] Happy path (mock adapter scripted): events arrive in order meta → deltas → usage → done; `contextRefs` are `{type, id}` objects, non-empty for a mini-express-app question ("how are users created" — a userService symbol/step ref appears within the top-3, matching 26's guarantee); assembled context ≤ 12,000 chars (captured prompt assert).
- [ ] Missing/wrong token → 401 before any SSE; bad Origin already 403 (33 regression); `Content-Type: text/plain` → 415; malformed body → 400; bad history alternation → 400 `invalid-history`; 3rd concurrent stream → 429 `busy`.
- [ ] Boost mapping: request with stepId boosts exactly that step + its excerpt (constructed near-tie flips — reuses 26's boost test pattern).
- [ ] Session budget: cap set to fit one answer → second request 429 `budget_exceeded`; `/api/session` reflects the spent total.
- [ ] Client disconnect mid-stream → provider AbortSignal fired within 100 ms (mock asserts).
- [ ] Provider error before any SSE byte → 502 JSON; after `meta` (before first delta) → in-band `error` event and clean stream end; no unhandled rejections.
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
