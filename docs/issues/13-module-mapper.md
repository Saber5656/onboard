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

- `src/analyze/modules/index.ts` — `mapModules({ files, manifest, edges, entryPoints }, logger) → { modules: ModuleInfo[], warnings }`.
- `src/analyze/modules/roles.ts` — role table + classifier.
- Unit tests on fixtures.

## Detailed Requirements

1. Candidates: depth-1 directories under root; depth-2 directories under root and under
   `src/`; plus workspace package roots (`manifest.workspaces` glob-expanded against
   directories present in files). Files directly in root belong to module `"."`.
2. Folding rules (§6.5 item 2): candidate with < 3 files merges into parent; a parent and
   child both survive only when the child has ≥ 8 files; after folding, if module count
   > 30, iteratively merge the smallest depth-2 modules into parents until ≤ 30
   (warning `modules-folded`).
3. Role classification: apply the §6.5 dir-name table on the module's basename
   (case-insensitive, exact match against the listed names); else `entry` when the
   module contains an EntryPoint file; else `unknown`.
4. Metrics: `fileCount`; `loc` = sum of line counts of read files (binary/oversized count
   0); module-level `fanIn`/`fanOut` = count of distinct other-modules with file edges
   into/out of this module; `topSymbols` = up to 5 exported symbol names from files in
   the module, ranked by their file's fanIn (file-level incoming edge count), tie
   (file, startLine) — requires symbols input: **add `symbols` to the function inputs**
   (coordinate signature with issue 10 output).
5. `ModuleInfo.id` = `"m" + short8(sha256(path))`; output sorted by path byte order.
6. Every file maps to exactly one module (the deepest surviving module whose path
   prefixes it; root files → `"."`) — invariant asserted in tests.

## Acceptance Criteria

- [ ] mini-express-app modules include `src/routes` (role http-api), `src/services` (role domain), `src/db` (role data-access), `src/util` (role shared-utils), `test` (role tests); `src/server.ts` lands in a module classified `entry` or the `"."`/`src` module contains the entry (assert actual folding result and freeze it).
- [ ] fanIn/fanOut: `src/db` has fanIn ≥ 1 (from services) and fanOut 0 among project modules.
- [ ] topSymbols for `src/services` includes the userService exports, ≤ 5 names.
- [ ] Partition invariant: sum of module fileCounts = total analyzed files; no file in two modules (property test over all fixtures).
- [ ] plain-docs: applying the folding rules yields module `docs` (role docs) plus root module `"."` (role unknown — the root matches no dir-name rule). Freeze the resulting structure in the test with a comment citing §6.5; if folding merges `docs` into `"."` (< 3 files), freeze that outcome instead and note it.

## Validation

`pnpm --filter onboard-cli-placeholder test` (modules suite); double-run determinism
equality; reviewer sanity-checks the mini-express-app module list against the fixture
tree.

## Dependencies

06, 07, 11, 12 (+ symbols from 10 for topSymbols).

## Non-goals

Semantic clustering; import-centrality-based role inference (the §6.5 note "(b)" applies
only via entry-point containment in v1); per-module narration (18).

## Design References

DESIGN §6.5, §5.2 ModuleInfo, §7.2 (ranking consumer), §5.5 (widgets).
