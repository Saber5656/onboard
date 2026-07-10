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

1. SVG sized from the layout extents (`max x + 220`, `max y + 90` as viewBox), responsive
   container, horizontal scroll when narrow (never squash below readable node width).
2. Nodes: rounded rect + label (text node, ellipsized > 18 chars with `<title>` full
   name), role class, tabbable.
3. Edges: cubic curves from node right edge to target left edge; stroke width
   `1 + log2(weight)` capped at 4; arrowhead marker; edges render **under** nodes.
4. Interaction: focus/hover on a node highlights incident edges (class toggle) and dims
   others; Enter opens the same popover style as 30 (reuse the popover component —
   extract to `components/Popover.tsx` in this issue and refactor 30 to use it if 30
   merged first; coordinate in PR).
5. Degenerate cases: 0 edges → render nodes only; 0 nodes → null.
6. Text nodes only (no HTML sink).

## Acceptance Criteria

- [ ] Component tests with a fixture graph (5 nodes / 6 edges): node positions match layout coords; edge count correct; weight-3 edge is visibly thicker than weight-1 (stroke-width attribute assertion); focus on a node marks exactly its incident edges.
- [ ] Long module name ellipsized with `<title>` fallback.
- [ ] Keyboard: Tab reaches every node; Enter opens popover; Escape closes.
- [ ] Zero-node widget renders null.

## Validation

`pnpm --filter viewer test`; manual: dependencies step of the fixture site on `file://`,
both themes (screenshot in PR).

## Dependencies

28, 29 (slot), 30 (shared popover/colors — soft dependency, coordinate refactor).

## Non-goals

Force-directed/interactive layout; file-level edges; cycle visualization beyond the
generate-time break notes (narrated, not drawn).

## Design References

DESIGN §9.3, §5.5 dep-graph, §7.2 item 4 (layout semantics).
