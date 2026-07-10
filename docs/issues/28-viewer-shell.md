# Title

Viewer shell: boot, hash router, state store, navigation chrome

## Summary

Implement the viewer application skeleton in `packages/viewer/` per DESIGN §9.2–9.3:
data boot (inline JSON parse + schemaVersion check), hand-rolled hash router, reducer
store with localStorage persistence, serve-mode detection via `/api/session`, and the
navigation chrome (Header, TourList sidebar, ProgressBar, HelpOverlay, prev/next
controls) — with placeholder step content (29 fills it).

## Context

All other viewer issues (29–32, 35) mount inside this shell. It must run as a classic
IIFE from `file://`, `http://127.0.0.1`, and static hosting with identical behavior
(minus chat).

## Scope

- `packages/viewer/src/boot.ts` — parse `#onboard-data` (`JSON.parse` of `textContent`), schemaVersion guard (§5.7: higher major → full-screen error with both versions), expose typed `SitePayload`.
- `packages/viewer/src/router.ts` — hash routes `#/tour/<tourId>/step/<n>`, `#/map`, `#/about`; unknown → first available tour step 0; `navigate()` + popstate/hashchange handling.
- `packages/viewer/src/store.ts` — Preact context + `useReducer`; state = `{ route, progress, theme, locale, session }`; progress persisted to `localStorage["onboard:<configHash>:progress"]` (§9.2 shape), theme to `onboard:theme`; storage failures (private mode) silently degrade to memory.
- `packages/viewer/src/session.ts` — boot-time `fetch("/api/session")`; 200 → store token+chat config; any failure → `session: null` (chat hidden; subtle hint per §9.2). No retries.
- Components: `Header` (repo name, commit short-sha + dirty badge, theme toggle, locale toggle placeholder — real i18n in 32), `TourList` (available tours + progress; unavailable greyed with reason), `ProgressBar`, `HelpOverlay` (`?`), prev/next buttons.
- Keyboard handling per §9.3 (`←/→`, `j/k`, `g t`, `?`); focus moved to step heading on navigation.
- Component tests (vitest + @testing-library/preact) with a fixture SitePayload.

## Detailed Requirements

1. No network requests other than the single `/api/session` probe (CSP already blocks
   externals; do not add any). No `eval`, no `dangerouslySetInnerHTML` in this issue's
   components (the two audited sinks arrive in 29).
2. Router is the only writer of `location.hash`; store mirrors it; step navigation
   updates progress (`lastStep`, `done` when final step visited).
3. `SitePayload` types imported from a shared `packages/viewer/src/types.ts` — hand-copied
   minimal interfaces matching §5.3 site payload (viewer package must not depend on the
   CLI package at runtime; a comment links both definitions; drift is caught by the 37
   e2e which loads real emitted data).
4. Boot failure modes (missing element, JSON parse error, schema mismatch) render a
   plain-text error screen with no repo-derived HTML.
5. Bundle constraints re-asserted: IIFE single chunk, no dynamic import, works when
   opened via `file://` (manual check + Playwright `file://` load in 37).
6. Accessibility basics now (full pass in 32): landmarks (`nav`, `main`), focus ring
   visible, `aria-current="step"` on the active step in TourList.

## Acceptance Criteria

- [ ] Component tests: boot renders tour list from fixture payload; unknown hash normalizes; prev/next and `j/k` navigate and persist progress (mock localStorage asserts writes); `?` toggles help; theme toggle flips `data-theme` attribute and persists.
- [ ] schemaVersion 2 payload → error screen (no crash, no partial render).
- [ ] `/api/session` mocked 404 → chat hint rendered, no chat UI, no console errors.
- [ ] Session mock 200 with `chat.enabled: true` → store carries token (chat panel itself lands in 35; assert store state only).
- [ ] `pnpm --filter viewer build` emits `app.js` (IIFE, no top-level `import`) + `app.css` into `packages/onboard/assets/viewer/`.

## Validation

`pnpm --filter viewer test && pnpm --filter viewer build`; then regenerate
mini-express-app site (27) and open from `file://`: tour list + navigation work with
placeholder step pane (manual smoke recorded in PR; automated in 37).

## Dependencies

01 (viewer package scaffold). Data contract: 27 (payload shape — develop against the
fixture payload checked in with this issue).

## Non-goals

Step/code rendering (29), widgets (30, 31), full i18n/a11y (32), chat UI (35).

## Design References

DESIGN §9.2–9.3, §5.7, §9.4 (basics), ADR-004.
