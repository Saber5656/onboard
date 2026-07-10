# Title

Viewer repo-map treemap widget

## Summary

Implement the `RepoMap` component per DESIGN §9.3 and §5.5: render the precomputed
`module-map` treemap widget as SVG (rect per module, role-tinted, loc-labeled), with
keyboard-accessible module selection opening a detail popover (role, metrics,
topSymbols), and focus highlighting for the step's `focus` list.

## Context

The layout is computed at generate time (18) — this component only maps rects 0..1000
coordinate space to responsive SVG. Keeping the viewer dumb preserves determinism and
makes this issue purely presentational.

## Scope

- `packages/viewer/src/components/RepoMap.tsx`.
- `packages/viewer/src/components/Popover.tsx` — the shared, accessible popover (labelled dialog, Escape/blur/outside-click close, viewport flip) created here and reused by 31.
- Role→color mapping in `app.css` (CSS variables per role, both themes; color-blind-safe distinctions — pair color with a role label, never color alone).
- Component tests.

## Detailed Requirements

1. Data contract: `RepoMap` receives the `module-map` widget payload (rects: §5.5)
   **plus** `modulesById` built from `payload.bundle.index.modules` (path, fileCount,
   fanIn/fanOut, topSymbols live there, not in the widget); a treemap id missing from
   the index renders the popover with only the widget-known fields (name, role, loc).
2. SVG `viewBox="0 0 1000 1000"`, `preserveAspectRatio="xMidYMid meet"`, responsive
   width (container-bound, max-height 60vh).
3. Rect per treemap entry: role CSS class; label = module name, rendered when the rect
   is wide enough (> 90 units) else exposed via `<title>` (native tooltip) and
   aria-label; loc shown as secondary label when height > 60 units.
4. `focus` module ids get a highlight class (thicker stroke + full opacity; non-focused
   rects dim to 70% when a focus list is non-empty).
5. Interaction: rects are `<a role="button" tabindex="0">`-equivalent (keyboard
   Enter/Space + click) → popover with name, path, role, fileCount, loc, fanIn/fanOut,
   topSymbols (all text nodes); Escape/blur/outside-click closes; one popover at a
   time; popover flips/clamps to stay within the viewport.
6. No text from the payload is ever injected as HTML (text nodes only — no sink here).
7. Role styling: role label always rendered as text in the popover and as the rect's
   accessible name suffix — color is never the only distinction; role CSS variables
   defined for both themes in `app.css`.
8. Empty/degenerate widget (0 modules) renders nothing (step remains narration-only).

## Acceptance Criteria

- [ ] Component tests with a fixture widget (6 modules) + matching index modules: 6 rects with correct x/y/w/h attributes; focus list dims others; Enter **and** Space **and** click each open the popover showing path, fileCount, fanIn/fanOut, and topSymbols joined from the index; Escape, blur, and outside-click close it; opening a second popover closes the first.
- [ ] Missing-index-entry case: popover shows widget-known fields only, no throw.
- [ ] Hostile strings: module `name`/`path`/`topSymbols` containing `<img src=x onerror=alert(1)>` render as text (no `img` element); `RepoMap.tsx` contains no `dangerouslySetInnerHTML` (lint rule from 29 applies).
- [ ] Narrow rect renders `<title>` fallback instead of visible label; role label text present in the popover and accessible name.
- [ ] Zero-module widget renders null.
- [ ] Axe-style checks (vitest-axe): rects reachable by Tab, popover labelled by module name; viewport flip logic covered by a positioned fixture or a documented manual check in the PR.

## Validation

`pnpm --filter viewer test`; manual: fixture site repo-map step on `file://` in both
themes (screenshot in PR).

## Dependencies

28 (shell), 29 (widget slot registration).

## Non-goals

Zoom/pan; file-level treemap drill-down (v2); layout computation (18).

## Design References

DESIGN §9.3, §5.5 module-map, §9.4 (a11y expectations), §7.2 item 2.
