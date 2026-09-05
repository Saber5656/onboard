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

- `packages/viewer/src/types.ts` — the exact payload contract (§9.1):
  `interface SitePayload { bundle: TourBundle; renderedSteps: Record<string, string>; highlightedExcerpts: Record<string, string> }` (hand-copied minimal `TourBundle` interfaces; a comment links to the CLI-side definitions; drift is caught by the 37 e2e on real emitted data).
- `packages/viewer/test/fixtures/site-payload.ts` — checked-in fixture payload used by all viewer tests (28–32, 35) so the viewer track never depends on running the generator.
- `packages/viewer/src/boot.ts` — parse `#onboard-data` (`JSON.parse` of `textContent`), schemaVersion guard (§5.7: higher major → full-screen error with both versions), expose the typed `SitePayload`.
- `packages/viewer/src/router.ts` — hash routes `#/tour/<tourId>/step/<n>`, `#/map`, `#/about`. Normalization rules (exact): unknown/malformed hash or unknown tourId → first available tour, step 0; known tour with out-of-range/malformed step → clamp to [0, last]; unavailable tour id → first available tour, step 0; zero available tours → `#/about` with an empty-state screen. `hashchange` is the **single canonical** navigation listener (no popstate handler; one reducer dispatch per navigation — tested).
- `packages/viewer/src/store.ts` — Preact context + `useReducer`; state = `{ route, progress, theme, locale, session }`; progress persisted to `localStorage["onboard:<configHash>:progress"]` (§9.2 shape); theme stored as `"auto" | "light" | "dark"` (default `auto`), cycle auto→light→dark→auto, `data-theme` always set to the **resolved** light/dark (auto resolves via `matchMedia("(prefers-color-scheme: dark)")`); storage failures (private mode) silently degrade to memory.
- `packages/viewer/src/session.ts` — boot-time `fetch("/api/session")`; three states: 200 + `chat.enabled: true` → enabled with token/budgets; 200 + `chat.enabled: false` → disabled with its `reason` rendered in the chat hint; any network failure/404 → `session: null` (static hosting; generic hint per §9.2). No retries.
- `StepShell` placeholder — a step pane with an `<h2 tabIndex={-1}>` heading that navigation focuses (29 replaces the body, keeping the heading contract).
- Components: `Header` (repo name, commit short-sha + dirty badge, theme toggle, locale toggle placeholder — real i18n in 32), `TourList` (available tours + progress; unavailable greyed with reason), `ProgressBar`, `HelpOverlay` (`?`), prev/next buttons.
- Keyboard handling per §9.3: `←/→` and `j/k` navigate; `g` arms a 1-second chord and a following `t` moves focus to the active TourList item (Escape or timeout cancels the chord); `?` toggles help. Navigation focuses the StepShell heading.
- Component tests (vitest + @testing-library/preact) with the fixture SitePayload.

## Detailed Requirements

1. No network requests other than the single `/api/session` probe (CSP already blocks
   externals; do not add any). No `eval`, no `dangerouslySetInnerHTML` in this issue's
   components (the two audited sinks arrive in 29).
2. Router is the only writer of `location.hash`; store mirrors it; step navigation
   updates progress (`lastStep`, `done` when final step visited).
3. The viewer package must not depend on the CLI package at runtime (types are the
   hand-copied contract in `types.ts`; §9.1 is the source of truth).
4. Boot failure modes (missing element, JSON parse error, schema mismatch) render a
   plain-text error screen with no repo-derived HTML.
5. Bundle constraints re-asserted: IIFE single chunk, no dynamic import, works when
   opened via `file://` (manual check + Playwright `file://` load in 37).
6. Accessibility basics now (full pass in 32): landmarks (`nav`, `main`), focus ring
   visible, `aria-current="step"` on the active step in TourList.

## Acceptance Criteria

- [ ] Component tests: boot renders tour list from the fixture payload; route matrix (unknown hash, unknown tour, out-of-range step, unavailable tour, zero-tours payload) normalizes per the exact rules; prev/next and `j/k` navigate, persist progress, and move focus to the StepShell heading; one reducer dispatch per navigation; `g t` chord focuses the TourList (and cancels on timeout/Escape); `?` toggles help.
- [ ] Theme: default `auto` resolves via matchMedia mock; cycle auto→light→dark→auto; `data-theme` always resolved value; persisted as the stored tri-state.
- [ ] Hostile chrome strings: repo name / tour titles / reasons containing `<img src=x onerror=alert(1)>` render as text only (no `img` element anywhere in the DOM).
- [ ] schemaVersion 2 payload → error screen (no crash, no partial render).
- [ ] Session states: mocked 404 → generic hint, no chat UI; mocked `enabled: false` with reason → the reason text renders; mocked `enabled: true` → store carries token (panel itself lands in 35); no console errors in any case.
- [ ] `pnpm --filter viewer build` emits `app.js` (IIFE, no top-level `import`) + `app.css` into `packages/onboard/assets/viewer/`.

## Validation

`pnpm --filter viewer test && pnpm --filter viewer build` — all against the checked-in
fixture payload (no generator run needed). Integration with a real emitted site is
gate 37's job.

## Dependencies

01 (viewer package scaffold). The payload contract is §9.1 (emitter 27 produces it;
this issue codifies it in `types.ts` + fixture — the two meet in gate 37).

## Non-goals

Step/code rendering (29), widgets (30, 31), full i18n/a11y (32), chat UI (35).

## Design References

DESIGN §9.2–9.3, §5.7, §9.4 (basics), ADR-004.
