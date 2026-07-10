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
- Role→color mapping in `app.css` (CSS variables per role, both themes; color-blind-safe distinctions — pair color with a role label, never color alone).
- Component tests.

## Detailed Requirements

1. SVG `viewBox="0 0 1000 1000"`, `preserveAspectRatio="xMidYMid meet"`, responsive
   width (container-bound, max-height 60vh).
2. Rect per treemap entry: role CSS class; label = module name, rendered when the rect
   is wide enough (> 90 units) else exposed via `<title>` (native tooltip) and
   aria-label; loc shown as secondary label when height > 60 units.
3. `focus` module ids get a highlight class (thicker stroke + full opacity; non-focused
   rects dim to 70% when a focus list is non-empty).
4. Interaction: rects are `<a role="button" tabindex="0">`-equivalent (keyboard Enter/
   Space + click) → popover with name, path, role, fileCount, loc, fanIn/fanOut,
   topSymbols (all text nodes); Escape/blur closes; one popover at a time; popover is
   positioned within the viewport (simple flip logic).
5. No text from the payload is ever injected as HTML (text nodes only — no sink here).
6. Empty/degenerate widget (0 modules) renders nothing (step remains narration-only).

## Acceptance Criteria

- [ ] Component tests with a fixture widget (6 modules): 6 rects with correct x/y/w/h attributes (scaled 1:1 in viewBox units); focus list dims others; keyboard Enter opens popover with the module's metrics; Escape closes and returns focus to the rect.
- [ ] Narrow rect renders `<title>` fallback instead of visible label.
- [ ] Zero-module widget renders null.
- [ ] Axe-style checks (vitest-axe or manual checklist in PR): rects reachable by Tab, popover labelled by module name.

## Validation

`pnpm --filter viewer test`; manual: fixture site repo-map step on `file://` in both
themes (screenshot in PR).

## Dependencies

28 (shell), 29 (widget slot registration).

## Non-goals

Zoom/pan; file-level treemap drill-down (v2); layout computation (18).

## Design References

DESIGN §9.3, §5.5 module-map, §9.4 (a11y expectations), §7.2 item 2.
