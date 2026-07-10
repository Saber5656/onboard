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

1. Step sequence exactly §7.2 (welcome; repo map; N module steps; dependencies; where
   next). Narration via StringTable keys (`arch.welcome`, `arch.map`, `arch.module`,
   `arch.deps`, `arch.next` — coordinate exact key set with issue 22; framework stub
   table until 22 lands).
2. Welcome facts: repoName, README title/firstParagraph (when present), stats (file
   count, loc, top-3 languages), list of available tours.
3. Module ranking (§7.2 item 3): `0.3·norm(fileCount) + 0.3·norm(loc) + 0.4·norm(fanIn)`
   where `norm(x) = x / max(x over candidate modules)` (max 0 ⇒ term 0); candidates
   exclude roles {tests, docs, build}; take top `min(8, candidates)`; tie-break module
   path asc. Excluded-role modules are mentioned in aggregate in the map step narration
   facts (counts only).
4. Module step: role, metrics (fileCount, loc, fanIn/fanOut), topSymbols, representative
   excerpt = declaration of the module's first topSymbol (fall back: file-head excerpt of
   the module's highest-fanIn file; no readable file → no excerpt, narration-only step).
   Anchor set to the excerpt span.
5. Treemap (strip algorithm, §7.2 item 2): input = ranked modules (all roles, not just
   top-8) sorted by loc desc; rows of ≤ 4 items; coordinate space 0..1000×1000; row
   height = round(1000 · rowLoc / totalLoc) (last row absorbs rounding remainder);
   within a row, widths ∝ loc with the same remainder rule. Output rects must tile the
   space exactly (property test: no overlap, full coverage within rounding of ±1 per
   edge, all integers).
6. Dep-graph layout (§7.2 item 4): nodes = top-8 modules + any module with an edge to/from
   them; edges = module-level import edges with weight = file-edge count. Layering:
   topological depth over the module DAG; cycles broken by removing the lowest-weight
   edge in each cycle (deterministic: iterate edges sorted by (weight asc, from, to)),
   removed edges noted in the deps-step facts. Coordinates: `x = depth·220`,
   `y = slot·90`, slot = index within layer sorted by module id.
7. Dependencies step narration facts: the 3 heaviest edges (weight desc, tie (from,to))
   with human labels.
8. Where-next step: lists other tours from availability with their reasons when
   unavailable.

## Acceptance Criteria

- [ ] mini-express-app: tour has 5 + N steps in the §7.2 order; module steps include `src/routes`, `src/services`, `src/db` (freeze exact set); each module step has anchor + excerptId (except narration-only fallbacks).
- [ ] Treemap property test (random module sets, 100 cases): integer rects, pairwise non-overlapping, union area = 1000×1000 ± rounding tolerance ≤ 4·rows.
- [ ] Dep-graph: mini-express-app layout places `src/server`-containing module at depth 0 and `src/db` at max depth (assert relative depths, not absolute pixels); cycle fixture (a↔b snippet project) breaks deterministically (same removed edge both runs).
- [ ] plain-docs: tour still builds with welcome + map + ≥ 1 module step + next (no dep edges → dependencies step omitted; assert the §7.2 sequence contract explicitly documents the omission rule: dependencies step only when ≥ 1 module edge exists).
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
