# Title

Viewer i18n (en/ja), theming polish, accessibility pass

## Summary

Complete the viewer's §9.4 requirements: embedded en/ja UI string table with runtime
toggle (narration language stays as generated, with the explanatory note), dark/light
theme completion across all components, and the accessibility gate — keyboard
reachability, focus management, landmarks/aria, contrast ≥ 4.5:1, reduced-motion
support.

## Context

§9.4 declares these a v1 gate, not polish. This issue lands after the visual components
(29–31) so the pass covers the real surface.

## Scope

- `packages/viewer/src/i18n/strings.ts` — typed UI string table (`en`, `ja`), exhaustive-key parity like issue 22 (compile-checked); keys for all chrome text (nav labels, buttons, hints, errors, chat placeholders for 35).
- Locale plumbing: store field (28) + Header toggle; default from `bundle.meta.locale`; persisted `localStorage["onboard:locale"]`; the "narration is generated in <locale>" note shown when UI locale ≠ meta.locale.
- Theme completion: `data-theme` variables audit across all components; `prefers-color-scheme` initial value when no stored preference; `prefers-reduced-motion` disables transitions.
- A11y audit + fixes across 28–31 components per the checklist below.
- Tests: string-table parity (type-level), locale toggle behavior, axe automated checks (vitest-axe) on the main views.

## Detailed Requirements

1. UI strings: no hard-coded user-facing literals left in components (grep-style test
   over built bundle for a canary set; code review for the rest). Dates/counts through
   the invariant helpers mirrored from §8.1 (viewer-local copies — ASCII digits in ja too).
2. Locale toggle affects UI chrome only; document the §9.4 note verbatim in both
   languages in the table.
3. A11y checklist (each verified by test or documented manual check in the PR):
   - every interactive element keyboard-operable with visible focus (28–31 + toggles);
   - focus moves to step heading on navigation (28's behavior — re-verify after 29);
   - landmarks: `banner` (Header), `nav` (TourList), `main` (StepView); skip-link to main;
   - `aria-current="step"`, popovers labelled, HelpOverlay is a labelled dialog with focus trap + Escape;
   - contrast ≥ 4.5:1 for text in both themes (checked with a contrast script over the CSS variable palette — include the script in `packages/viewer/scripts/contrast-check.mjs`, run in CI test step);
   - `prefers-reduced-motion: reduce` → no animated transitions (CSS media query).
4. No new dependencies except `vitest-axe` (dev).

## Acceptance Criteria

- [ ] String tables: both locales implement every key (compile-time); toggling locale re-renders chrome in ja incl. the narration-language note; preference persists across reload (test with mocked storage).
- [ ] vitest-axe: no violations of severity serious/critical on TourList, StepView (with fixture content), RepoMap, DepGraph, HelpOverlay.
- [ ] Contrast script passes for both theme palettes (CI-executed).
- [ ] Reduced-motion: transition durations are 0 under the media query (computed-style assertion).
- [ ] Manual keyboard walkthrough (documented in PR): full tour navigation without a pointer.

## Validation

`pnpm --filter viewer test && pnpm --filter viewer build`; manual keyboard + screenshots
(both themes × both locales) attached to the PR.

## Dependencies

28, 29, 30, 31.

## Non-goals

Narration translation at runtime (regenerate to change — §9.4); locales beyond en/ja;
RTL support.

## Design References

DESIGN §9.4, §9.2 (persistence keys), §8.1 (formatting invariants).
