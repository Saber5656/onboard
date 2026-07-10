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
- Context chips from `meta.contextRefs`: `step:` refs navigate via router; `x:`/`sym:`/`mod:` refs navigate to the step containing them when resolvable from the payload, else render as non-link chips.
- Token display: per-answer `~N tokens` line from `usage`; session meter (used/budget) updated from `sessionTotal`; budget-exhausted state per §10.3.
- UI strings via the i18n table (32); keyboard: `c` toggles, Escape closes, input submit on Enter (Shift+Enter newline).
- Component tests with a mocked fetch/SSE.

## Detailed Requirements

1. Mount condition: `session.chat.enabled` only (28's store); hidden entirely otherwise
   (the §9.2 hint outside this component covers discovery).
2. Requests: `POST /api/chat` with `X-Onboard-Token`, question, current `tourId`/`stepId`
   from the route, history = last ≤ 8 turns (client-trimmed).
3. Streaming render: deltas appended to the live message (raf-batched); "stop" button
   aborts (client-side AbortController) and marks the message truncated.
4. Security rendering rules (§11.5): no markdown rendering, no links from model text,
   no HTML — the fence parser is the only structure; test includes an answer containing
   `<img src=x onerror=…>` and asserts it renders as literal text.
5. Error states: 401/403 → "session expired — restart serve" style message; 409 →
   disabled-with-reason; 429 budget → persistent budget-spent banner (§10.3);
   in-band `error` event → inline system message with `code`.
6. History and draft survive panel toggle (component state in store), not page reload
   (no persistence — session is per-serve-process anyway).
7. A11y: panel is a labelled complementary landmark; new-message announcements via
   `aria-live="polite"`; focus management on open/close; meets 32's checklist.

## Acceptance Criteria

- [ ] Mock SSE round-trip renders: chips (with a `step:` chip that navigates on click), streamed text growing, code fence as `<pre><code>`, token line, updated session meter.
- [ ] XSS answer fixture renders as escaped literal text (innerHTML assertion: no `img` element exists).
- [ ] Budget 429 shows the banner and disables input; 409 no-api-key shows reason.
- [ ] Stop button aborts the fetch (mock asserts abort) and the message is marked truncated.
- [ ] `c`/Escape keyboard behavior; vitest-axe on the open panel: no serious/critical violations.

## Validation

`pnpm --filter viewer test`; end-to-end against a real server uses the env-gated mock
seam (`ONBOARD_TEST_MOCK_LLM=1`, defined by issue 37) in the Playwright serve smoke —
the config enum deliberately has no `mock` provider (issue 24). Manual real-provider
smoke is deferred to issue 41.

## Dependencies

32 (i18n/a11y base), 34 (endpoint), 28 (store/session).

## Non-goals

Markdown/link rendering of answers; chat history persistence; BYO-key input of any kind
(ADR-003); mobile layout beyond basic responsiveness.

## Design References

DESIGN §10.7, §10.3 (protocol/errors), §11.5 (chat row), §1.2 (visible tokens).
