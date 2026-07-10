# Title

Architecture tour builder with treemap and dep-graph layouts

## Summary

Implement `src/tours/architecture.ts` per DESIGN §7.2: the always-available tour —
welcome step, repo-map step with a deterministically laid-out treemap widget, one step
per top-ranked module with a representative excerpt, a dependencies step with a layered
dep-graph widget, and a closing step pointing to the other available tours.

## Context

This is the first tour every user sees and the only one guaranteed on every repo. It
also owns the two precomputed widget layouts (§5.5) that viewer issues 30/31 render.

## Scope

- `src/tours/architecture.ts` — the builder (framework contract, 17).
- `src/tours/layout/treemap.ts` — strip treemap layout (pure).
- `src/tours/layout/layered.ts` — layered DAG layout (pure).
- Unit tests + layout property tests.

## Detailed Requirements

1. Step sequence exactly §7.2 — welcome; repo map; N module steps; dependencies (**only
   when `model.moduleEdges` is non-empty** — §7.2 rule; docs-only repos omit it); where
   next. Narration via StringTable keys (`arch.welcome`, `arch.map`, `arch.module`,
   `arch.deps`, `arch.next` — key set fixed by 17/22).
2. Welcome facts: repoName, `model.readme` title/firstParagraph (when present), stats
   (file count, loc, top-3 languages), list of available tours.
3. Module ranking (§7.2 item 3): `0.3·norm(fileCount) + 0.3·norm(loc) + 0.4·norm(fanIn)`
   where `norm(x) = x / max(x over candidate modules)` (max 0 ⇒ term 0); candidates
   exclude roles {tests, docs, build}; take top `min(8, candidates)`; tie-break module
   path asc. Excluded-role modules are mentioned in aggregate in the map step narration
   facts (counts only).
4. Module step: role, metrics (fileCount, loc, fanIn/fanOut), topSymbols, representative
   excerpt — lookup: the module's first `topSymbols` name resolved against
   `model.symbols` filtered to `module.files`, first match by (file, startLine);
   fallback: file-head excerpt of the module's highest-fanIn file (per-file fanIn =
   incoming count in `model.imports.edges`, tie path asc, only `lang !== null` files);
   no readable file → no excerpt, narration-only step. Anchor = the returned excerpt
   span (17's `takeExcerpt` return).
5. Treemap (strip algorithm, §7.2 item 2): input = **all** `model.modules` sorted by
   loc desc, tie path asc; when every module has loc 0, weight each module as 1
   (equal-area tiles). Rows of ≤ 4 items; coordinate space 0..1000×1000; integer
   coordinates with exact tiling: within a row the **last item absorbs the width
   remainder**, and the **last row absorbs the height remainder** — union is exactly
   1000×1000, no overlaps, no tolerance.
6. Dep-graph layout (§7.2 item 4): nodes = top-8 modules + any module with a
   `model.moduleEdges` edge to/from them; edges from `model.moduleEdges` (weight =
   file-edge count, produced by 13). Cycle handling (deterministic, §7.2): iterate
   edges sorted by (weight desc, from asc, to asc), keep an edge only if it does not
   create a cycle in the kept-edge DAG (DFS/union-find check); skipped edges are
   recorded in the deps-step facts. Layering: topological depth over the kept DAG;
   `x = depth·220`, `y = slot·90`, slot = index within layer sorted by module id.
7. Dependencies step narration facts: the 3 heaviest edges (weight desc, tie
   (from, to) asc) with human labels, plus skipped-cycle-edge notes when any.
8. Where-next step: lists other tours from availability — available ones by title,
   unavailable ones with their `reason`.

## Acceptance Criteria

- [ ] mini-express-app: tour has the §7.2 step order; module steps include `src/routes`, `src/services`, `src/db` (freeze exact set); each module step has anchor + excerptId (except narration-only fallbacks); the deps-step facts list the 3 heaviest edges in (weight desc, from, to) order with human labels.
- [ ] Treemap property test (random module sets incl. an all-zero-loc set, 100 cases): integer rects, pairwise non-overlapping, union area exactly 1,000,000, bounding box exactly 1000×1000.
- [ ] Dep-graph: mini-express-app layout places the `src/server`-containing module at depth 0 and `src/db` at max depth (relative depths); cycle fixture (a↔b snippet project) keeps/skips the same edge on both runs, and the skipped edge appears in the deps facts.
- [ ] plain-docs: tour builds with welcome + map + ≥ 1 module step + where-next; the dependencies step is absent (moduleEdges empty); where-next facts include the unavailable entry-flow tour with its reason.
- [ ] Double-run determinism on serialized tour.

## Validation

`pnpm --filter onboard-cli-placeholder test` (architecture suite). Reviewer renders
nothing — layout correctness is property-tested; eyeball the frozen step list.

## Dependencies

17 (framework), 13 (modules), 11 (edges), 10 (symbols).

## Non-goals

Widget rendering (30, 31); narration prose (22); LLM enhancement (25).

## Design References

DESIGN §7.2, §5.5 (widget shapes/layout rules), §5.3, §6.5.
