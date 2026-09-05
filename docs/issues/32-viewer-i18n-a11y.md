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

- `packages/viewer/src/i18n/strings.ts` — typed UI string table (`en`, `ja`),
  exhaustive-key parity like issue 22 (compile-checked). Keys cover all chrome text
  (nav labels, buttons, hints, errors, empty states) **and the complete chat key set
  consumed by 35** (fixed here because 35 depends on this issue): `chat.placeholder`,
  `chat.send`, `chat.stop`, `chat.truncated`, `chat.contextLabel`, `chat.tokensLine`,
  `chat.sessionMeter`, `chat.budgetSpent`, `chat.disabledNoConfig`,
  `chat.disabledNoApiKey`, `chat.errorGeneric`, `chat.hintServe`.
- Locale plumbing: store field (28) + Header toggle; default from `bundle.meta.locale`;
  persisted per-bundle as `localStorage["onboard:<configHash>:locale"]` (§9.2 —
  precedence: stored override > meta.locale). The narration-language note is shown when
  UI locale ≠ meta.locale, exact strings fixed here:
  en `"Narration was generated in {locale}. Regenerate with --locale to change it."`,
  ja `"ナレーションは{locale}で生成されています。変更するには --locale を付けて再生成してください。"`.
- Theme completion: audit `data-theme` variable coverage across all components (the
  tri-state auto/light/dark model itself ships in 28); `prefers-reduced-motion`
  disables transitions.
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
   - contrast ≥ 4.5:1 in both themes, checked by `packages/viewer/scripts/contrast-check.mjs`
     over an **explicit token-pair list** (body text/background, muted text/background,
     accent-on-background, focus ring vs adjacent surface, role label text on each role
     tile fill, code foreground/background from the Shiki theme variables) — palette
     pairs are the gate; rendered-component spot-checks go in the PR screenshots;
   - `prefers-reduced-motion: reduce` → no animated transitions, validated by parsing
     `app.css` for the media rule that zeroes transition durations (jsdom cannot
     evaluate media-dependent computed styles; the behavioral check runs in 37's
     Playwright smoke with reduced-motion emulation).
4. No new dependencies except `vitest-axe` (dev).

## Acceptance Criteria

- [ ] String tables: both locales implement every key incl. the chat set (compile-time); toggling locale re-renders chrome in ja incl. the exact narration-language note; per-bundle preference persists across reload (mocked storage) and does not leak across different configHash values.
- [ ] vitest-axe: zero violations with axe impact `serious`/`critical` on TourList, StepView (fixture content), RepoMap, DepGraph, HelpOverlay; lower-impact findings are triaged in the PR (fix or documented exception).
- [ ] Contrast script passes for the enumerated token pairs in both themes (CI-executed).
- [ ] Reduced-motion: `app.css` contains the parsed `prefers-reduced-motion` rule zeroing transitions (behavioral check deferred to 37).
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
