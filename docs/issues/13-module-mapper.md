# Title

Module mapper: directory folding, role classification, metrics

## Summary

Implement `src/analyze/modules/` per DESIGN §6.5: fold the file tree into ≤ ~30
human-meaningful modules (directories), classify each with the role taxonomy, and attach
metrics (fileCount, loc, fanIn/fanOut aggregated from file import edges, topSymbols).

## Context

Modules are the vocabulary of the architecture tour (18), the treemap/dep-graph widgets,
the contributing tour's "where to start", and chat retrieval docs. Stable, sensible
folding matters more than cleverness.

## Scope

- `src/analyze/modules/index.ts` — `mapModules({ files, manifest, edges, entryPoints, symbols }, logger) → { modules: ModuleInfo[], warnings }`.
- `src/analyze/modules/roles.ts` — role table + classifier.
- Unit tests on fixtures.

## Detailed Requirements

1. Candidates: depth-1 directories under root; depth-2 directories under root and under
   `src/`; plus workspace package roots — expanded by matching `manifest.workspaces`
   glob patterns against the **directory-prefix set derived from sanitized
   `FileNode.path` values only** (no filesystem walk, no symlink resolution, results
   sorted byte-order). Files directly in root belong to module `"."`.
2. File→module assignment uses **segment-boundary matching**: a file belongs to
   candidate `m` iff `path === m` is impossible (files ≠ dirs) so:
   `path.startsWith(m + "/")`; among matches take the deepest; no match → `"."`
   (prevents `src/app` claiming `src/application/x.ts`).
3. Folding rules (§6.5 item 2), evaluated on **assigned file counts** bottom-up
   (deepest candidates first): a candidate with < 3 assigned files merges its files
   into its parent candidate (or `"."`); a surviving parent and child both remain only
   when the child has ≥ 8 assigned files (else child merges up). After folding, if
   module count > 30, iteratively merge the smallest surviving depth-2 module —
   "smallest" = (fileCount asc, loc asc, path byte-order asc) — into its parent until
   ≤ 30 (warning `modules-folded`).
4. Role classification: apply the §6.5 dir-name table on the module's basename
   (case-insensitive, exact match against the listed names — the table includes `util`);
   else `entry` when the module contains an EntryPoint file; else `unknown`.
5. Metrics: `fileCount` (assigned files); `loc` = Σ `FileNode.lineCount ?? 0` over
   assigned files (issue 06 provides lineCount; never re-read content here);
   module-level `fanIn`/`fanOut` = count of distinct other-modules with file edges
   into/out of this module; `topSymbols` = up to 5 exported symbol names from files in
   the module, ranked by their file's fanIn (file-level incoming edge count), tie
   (file, startLine).
6. `ModuleInfo.id` = `"m" + short8(sha256(path))`; output sorted by path byte order.
7. Partition invariant: every file maps to exactly one surviving module — asserted in
   tests (sum of fileCounts = total files; no overlaps).

## Acceptance Criteria

- [ ] mini-express-app modules include `src/routes` (role http-api), `src/services` (role domain), `src/db` (role data-access), `src/util` (role shared-utils), `test` (role tests); `src/server.ts` lands in a module classified `entry` or the `"."`/`src` module contains the entry (assert actual folding result and freeze it).
- [ ] fanIn/fanOut: `src/db` has fanIn ≥ 1 (from services) and fanOut 0 among project modules.
- [ ] topSymbols for `src/services` includes the userService exports, ≤ 5 names.
- [ ] Partition invariant: sum of module fileCounts = total analyzed files; no file in two modules (property test over all fixtures).
- [ ] plain-docs: `docs/` has 2 assigned files (< 3) so it merges up — the deterministic result is exactly one module `"."` with fileCount 4, role `unknown`; assert precisely this (comment cites the §6.5 fold rule).
- [ ] Segment-boundary test: `src/app/x.ts` vs `src/application/y.ts` snippet partition never cross-assigns; parent/child boundary tests at 2/3 and 7/8 assigned files freeze the fold outcomes.

## Validation

`pnpm --filter onboard-cli-placeholder test` (modules suite); double-run determinism
equality; reviewer sanity-checks the mini-express-app module list against the fixture
tree.

## Dependencies

06, 07, 10 (symbols for topSymbols), 11, 12.

## Non-goals

Semantic clustering; import-centrality-based role inference (the §6.5 note "(b)" applies
only via entry-point containment in v1); per-module narration (18).

## Design References

DESIGN §6.5, §5.2 ModuleInfo, §7.2 (ranking consumer), §5.5 (widgets).
