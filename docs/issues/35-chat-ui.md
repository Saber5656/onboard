# Title

Chat panel UI with token meter and context chips

## Summary

Implement `ChatPanel` per DESIGN §10.7: a right-side panel (toggle `c`) mounted only
when `/api/session` reports chat enabled — message list, SSE-streamed answers rendered
as plain text + fenced-code blocks (text nodes only), per-answer token line, session
budget meter, context chips linking to retrieved steps/excerpts, and inline error
states for every §10.3 error code.

## Context

The chat surface makes token consumption *visible* (product thesis §1.2) and is the
last rendering surface in the XSS contexts table (§11.5): LLM output must never become
HTML.

## Scope

- `packages/viewer/src/components/ChatPanel.tsx` + `chat-client.ts` (fetch + SSE parsing of §10.3 events, AbortController on unmount/stop).
- Rendering: parser that splits answer text on ``` fences → `<pre><code>` blocks (text nodes) and paragraph text (pre-wrap text nodes); nothing else interpreted (§10.7).
- Context chips from `meta.contextRefs` (`{ type: "step"|"excerpt"|"symbol"|"module", id }` objects, §10.3): `type: "step"` → navigate via router to that step; `type: "excerpt"` → navigate to the first step (tours in bundle order) whose `excerptId` equals the id, else non-link chip; `type: "symbol"` and `"module"` → always non-link chips in v1 (label only).
- Token display: per-answer `~N tokens` line from `usage`; session meter (used/budget) updated from `sessionTotal`; budget-exhausted state per §10.3.
- UI strings via the i18n table (32); keyboard: `c` toggles, Escape closes, input submit on Enter (Shift+Enter newline).
- Component tests with a mocked fetch/SSE.

## Detailed Requirements

1. Mount condition: `session.chat.enabled` only (28's store); hidden entirely otherwise
   (the §9.2 hint outside this component covers discovery).
2. Requests: `POST /api/chat` with `X-Onboard-Token` and `Content-Type:
   application/json`; question; current `tourId` from the route and `stepId` resolved
   as `payload.bundle.tours[<tourId>].steps[<n>].id` (the route carries the index, not
   the id); history = the last ≤ 8 **messages** (`{role, content}` items, excluding the
   new question), client-trimmed from the oldest side so alternation stays valid.
3. Streaming render: deltas appended to the live message (raf-batched); "stop" button
   aborts (client-side AbortController) and marks the message truncated.
4. Security rendering rules (§11.5): no markdown rendering, no links from model text,
   no HTML — the fence parser is the only structure.
5. Error-state table (complete, each with its i18n key from 32):
   | Status/event | UI state |
   |---|---|
   | 400 / 413 / 415 | inline system message `chat.errorGeneric` + code |
   | 401 / 403 | "session expired — restart serve" message; input stays enabled for retry after reload |
   | 409 | disabled state with the session `reason` text (`chat.disabledNoConfig` / `chat.disabledNoApiKey`) |
   | 429 busy | inline "try again in a moment" system message |
   | 429 budget_exceeded | persistent `chat.budgetSpent` banner; input disabled |
   | 502 | inline system message with code |
   | in-band `error` event | inline system message with the event's `code` |
6. History and draft survive panel toggle (component state in store), not page reload
   (no persistence — session is per-serve-process anyway).
7. A11y: panel is a labelled complementary landmark; new-message announcements via
   `aria-live="polite"`; focus management on open/close; meets 32's checklist.

## Acceptance Criteria

- [ ] Mock SSE round-trip renders: chips (a `step` chip navigates on click; an unresolvable `excerpt` chip and a `symbol` chip render as non-links), streamed text growing, code fence as `<pre><code>`, token line, updated session meter.
- [ ] XSS/link fixtures: an answer containing `<img src=x onerror=…>`, a raw `https://` URL, and `[x](javascript:alert(1))` renders entirely as literal text — no `img`, no `a` elements anywhere in the message list.
- [ ] Error-state table covered: one test per row (mocked responses/events), asserting the specified UI state.
- [ ] Draft + history persistence: type a draft, complete one turn, toggle the panel closed and open (`c`, Escape) → draft text and history remain (no localStorage writes for chat).
- [ ] Stop button aborts the fetch (mock asserts abort) and the message is marked truncated.
- [ ] `c`/Escape keyboard behavior; vitest-axe on the open panel: zero violations at axe impact serious/critical.

## Validation

`pnpm --filter viewer test`; end-to-end against a real server uses the env-gated mock
seam (`ONBOARD_TEST_MOCK_LLM=1`, defined by issue 37) in the Playwright serve smoke —
the config enum deliberately has no `mock` provider (issue 24). Manual real-provider
smoke is deferred to issue 41.

## Dependencies

32 (i18n/a11y base + chat string keys), 34 (endpoint). (28's store/session arrives
transitively through 32.)

## Non-goals

Markdown/link rendering of answers; chat history persistence; BYO-key input of any kind
(ADR-003); mobile layout beyond basic responsiveness.

## Design References

DESIGN §10.7, §10.3 (protocol/errors), §11.5 (chat row), §1.2 (visible tokens).
