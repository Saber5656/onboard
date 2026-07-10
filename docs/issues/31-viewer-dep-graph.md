# Title

Viewer dependency-graph widget

## Summary

Implement the `DepGraph` component per DESIGN §9.3 and §5.5: render the precomputed
layered `dep-graph` widget as SVG — module nodes at their generate-time coordinates,
weighted edges as curves with arrowheads, role tinting consistent with RepoMap, and
keyboard-accessible node inspection (highlight incident edges).

## Context

Companion widget to 30 for the architecture tour's dependencies step. Layout (layer x,
slot y) is precomputed (18); this component only draws and handles interaction.

## Scope

- `packages/viewer/src/components/DepGraph.tsx`.
- Shared role-color usage with 30 (same CSS variables).
- Component tests.

## Detailed Requirements

1. Geometry (exact, testable): node box = 180×48 units, top-left at the widget's
   (x, y); SVG viewBox = `0 0 (max x + 180 + 40) (max y + 48 + 40)`; responsive
   container with horizontal scroll when narrow (never squash below readable node
   width).
2. Nodes: rounded rect + label (text node, ellipsized > 18 chars with `<title>` full
   name), role class, tabbable.
3. Edges: cubic Bézier from source right-center `(x+180, y+24)` to target left-center
   `(tx, ty+24)` with control points offset ±60 on the x axis; stroke width
   `1 + log2(max(weight, 1))` capped at 4; arrowhead marker; edges render **under**
   nodes. Edges referencing an unknown node id are skipped without throwing.
4. Interaction: focus/hover on a node highlights incident edges (class toggle) and dims
   others; Enter opens a popover using the shared `components/Popover.tsx` from issue 30
   (hard dependency), with module metadata joined from `payload.bundle.index.modules`
   the same way as 30.
5. Degenerate cases: 0 edges → render nodes only; 0 nodes → null.
6. Text nodes only (no HTML sink).

## Acceptance Criteria

- [ ] Component tests with a fixture graph (5 nodes / 6 edges): node positions match layout coords (box rule); edge path endpoints match the center formula; weight-3 edge is thicker than weight-1 (stroke-width attribute assertion); focus on a node marks exactly its incident edges; a dangling edge is skipped silently.
- [ ] Popover content: node label + role from the widget, incident in/out edge counts computed locally, plus index-module metadata when the id resolves (same join as 30); missing index entry degrades gracefully.
- [ ] Hostile label `<img src=x onerror=alert(1)>` renders as text/title content only (no `img` element; no `dangerouslySetInnerHTML` in the file).
- [ ] Long module name ellipsized with `<title>` fallback.
- [ ] Keyboard: Tab reaches every node; Enter opens popover; Escape closes.
- [ ] Zero-node widget renders null.

## Validation

`pnpm --filter viewer test`; manual: dependencies step of the fixture site on `file://`,
both themes (screenshot in PR).

## Dependencies

28 (shell), 29 (widget slot registry), 30 (shared Popover + role colors).

## Non-goals

Force-directed/interactive layout; file-level edges; cycle visualization beyond the
generate-time break notes (narrated, not drawn).

## Design References

DESIGN §9.3, §5.5 dep-graph, §7.2 item 4 (layout semantics).
