# Title

Viewer step view and code pane

## Summary

Implement `StepView` and `CodePane` per DESIGN §9.3: render pre-rendered narration HTML
and pre-highlighted excerpt HTML through the two audited sinks, show step title/anchor
metadata (file path + line range, copy-path button), and wire widget slots (30/31 mount
into them; unknown widget types render nothing).

## Context

These are the only two components allowed to inject HTML (§11.5) — both inputs are
generate-time sanitized (27). Everything else here must be text-node rendering.

## Scope

- `packages/viewer/src/components/StepView.tsx` — step title (text), narration container (audited sink #1: `renderedSteps[stepId]`), widget slot list, prev/next footer.
- `packages/viewer/src/components/CodePane.tsx` — excerpt container (audited sink #2: `highlightedExcerpts[excerptId]`), header with `file:start–end` (text nodes), "copy path" button (`navigator.clipboard`, fallback to a selectable input), anchor-line emphasis (CSS class on the anchor range when the widget payload provides it — v1: highlight the whole excerpt block subtly; per-line emphasis only if Shiki output carries line elements, else skip — decide at implementation and document).
- Audited-sink lint rule: enable an ESLint restriction so `dangerouslySetInnerHTML` is only permitted in these two files (eslint flat-config override; violation elsewhere fails lint).
- Component tests.

## Detailed Requirements

1. Sink discipline: the two sinks read **only** from the payload maps by id; missing id →
   fallback text ("excerpt unavailable") — never construct HTML from other fields.
2. Step with `excerptId: null` renders narration-only layout (no empty code frame).
3. Copy-path copies the repo-relative POSIX path exactly (no absolute paths exist in the
   payload by design §5.1 — assert in test fixture).
4. Long excerpts: CodePane max-height with internal scroll; horizontal overflow scrolls
   (no wrap — code); both themes styled via the dual-theme CSS variables emitted by
   Shiki (27) — include the required CSS in `app.css` (documented selector contract:
   `.shiki`, `--shiki-dark` variables per Shiki dual-theme docs; verify exact mechanism
   at implementation, KU-6).
5. Keyboard: sinks' containers are not focus traps; step heading receives focus on
   navigation (28's contract) and the narration container is `tabindex="-1"`-reachable
   for screen readers.
6. Widget slot: `step.widgets[]` mapped to registered components by `type`; unknown type
   → `null` (forward-compat §5.7 additive rule).

## Acceptance Criteria

- [ ] Component tests: step renders narration HTML from fixture map (assert innerHTML equality); excerpt header shows `src/services/userService.ts:1–24`-style text; copy button writes to a mocked clipboard; null excerpt renders no code frame; unknown widget type renders nothing and logs nothing.
- [ ] Lint: adding `dangerouslySetInnerHTML` to Header fails `pnpm lint` (test the rule by fixture or documented manual check in PR).
- [ ] Missing excerptId in map → fallback text, no throw.
- [ ] Dark/light: excerpt container computed styles differ between `data-theme` values (smoke assertion on background variable).

## Validation

`pnpm --filter viewer test && pnpm lint`; manual: regenerate fixture site, open
`file://`, walk all mini-express-app steps — narration + code render, no console errors
(recorded in PR; automated in 37).

## Dependencies

28 (shell/store/router), 27 (payload contract).

## Non-goals

Widgets themselves (30, 31); chat (35); syntax highlighting logic (generate-time, 27);
per-line permalink UI (v2).

## Design References

DESIGN §9.3, §11.5 (sinks), §5.5 (widget slot), KU-6.
