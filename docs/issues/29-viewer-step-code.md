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

- `packages/viewer/src/components/StepView.tsx` — step title (text), narration container (audited sink #1: `renderedSteps[stepId]`; missing id → text fallback "step unavailable"), widget slot list, prev/next footer. Replaces 28's StepShell body while keeping its focusable-heading contract.
- `packages/viewer/src/components/CodePane.tsx` — excerpt container (audited sink #2: `highlightedExcerpts[excerptId]`; missing id → text fallback "excerpt unavailable"), header with `file:start–end` (text nodes), "copy path" button (`navigator.clipboard`, fallback to a selectable input), anchor emphasis from **`step.anchor`** (when `anchor.file` equals the excerpt file and the ranges overlap → emphasize; v1 = subtle whole-block emphasis; per-line only if the pinned Shiki output exposes line elements — decide at implementation and document in code).
- `packages/viewer/src/components/widgets/index.ts` — the widget slot registry: `registerWidget(type, Component)` + `WidgetSlot` resolving `step.widgets[]` by type (unknown → null). Issues 30/31 register their components here.
- Audited-sink lint rule: enable an ESLint restriction so `dangerouslySetInnerHTML` is only permitted in these two files (eslint flat-config override; violation elsewhere fails lint).
- Component tests.

## Detailed Requirements

1. Sink discipline: the two sinks read **only** from the payload maps
   (`renderedSteps`/`highlightedExcerpts`) by id — `step.body`, `step.title`,
   `excerpt.text`, and every other payload field are **never** injected as HTML
   anywhere (text nodes only).
2. Step with `excerptId: null` renders narration-only layout (no empty code frame).
3. Copy-path copies the repo-relative POSIX path exactly (no absolute paths exist in the
   payload by design §5.1 — assert in test fixture).
4. Long excerpts: CodePane max-height with internal scroll; horizontal overflow scrolls
   (no wrap — code); both themes styled via the dual-theme CSS variables emitted by
   Shiki (27) — include the required CSS in `app.css` (documented selector contract:
   `.shiki`, `--shiki-dark` variables per Shiki dual-theme docs; verify exact mechanism
   at implementation, KU-6).
5. Keyboard/a11y: sinks' containers are not focus traps; the step heading receives focus
   on navigation (28's contract); the narration container is programmatically focusable
   (`tabIndex={-1}`), associated with the heading via `aria-labelledby`, and excluded
   from the tab order.
6. Widget slot registry as scoped above; unknown type → `null` (forward-compat §5.7
   additive rule).

## Acceptance Criteria

- [ ] Component tests: step renders narration HTML from fixture map (assert innerHTML equality); excerpt header shows `src/services/userService.ts:1–24`-style text; copy button writes to a mocked clipboard; null excerpt renders no code frame; unknown widget type renders nothing and logs nothing; registered widget type mounts.
- [ ] Hostile sink tests: fixture where `step.body`, `step.title`, `excerpt.file`, and `excerpt.text` all contain `<img src=x onerror=alert(1)>` — only the two sink containers contain HTML (from the maps), every other field renders as text, and no `img` element exists.
- [ ] Fallbacks: missing `renderedSteps[stepId]` → "step unavailable" text; missing `highlightedExcerpts[excerptId]` → "excerpt unavailable" text; no throw in either case.
- [ ] Anchor emphasis: fixture step whose anchor overlaps the excerpt gets the emphasis class; non-overlapping anchor does not.
- [ ] Theme contract (jsdom-safe): excerpt container carries the pinned Shiki dual-theme class/variable selectors expected by `app.css` (visual verification happens in the 37 Playwright smoke, not here).
- [ ] Lint: adding `dangerouslySetInnerHTML` to Header fails `pnpm lint` (documented spot-check in PR).

## Validation

`pnpm --filter viewer test && pnpm lint`; manual: regenerate fixture site, open
`file://`, walk all mini-express-app steps — narration + code render, no console errors
(recorded in PR; automated in 37).

## Dependencies

28 (shell/store/router + payload fixture; the payload contract is §9.1 via 28's
`types.ts`).

## Non-goals

Widgets themselves (30, 31); chat (35); syntax highlighting logic (generate-time, 27);
per-line permalink UI (v2).

## Design References

DESIGN §9.3, §11.5 (sinks), §5.5 (widget slot), KU-6.
