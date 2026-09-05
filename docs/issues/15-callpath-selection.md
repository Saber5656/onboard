# Title

Call-path tracer (2/2): significance scoring and path selection

## Summary

Implement the selection half of DESIGN §6.10 in `src/analyze/flow/select.ts`: from each
entry point, repeatedly pick the most significant in-repo callee (normative scoring
table), follow it as the next hop up to `flow.maxDepth`, record skipped significant
siblings as `branchNotes`, and emit one linear `CallPath` per flow-eligible entry.

## Context

Turns raw resolutions (14) into the narrative spine of entry-flow tours (19). The
scoring heuristics are expected to need tuning (KU-5) — they must therefore live in one
table with exhaustive unit coverage, not be scattered.

## Scope

- `src/analyze/flow/select.ts` — `selectCallPaths({ entryPoints, ts, files, modules, symbols }, config, logger) → { callPaths: CallPath[], warnings }` (modules = issue 13 partition for same-module scoring; symbols = issue 10 index).
- `src/analyze/flow/score.ts` — significance scoring (pure function + constant table) and `countBodyStatements(target, ts)` helper.
- Unit tests.

## Detailed Requirements

1. For each `EntryPoint` with `flowEligible === true` (§5.2 field, set by issue 12):
   start at (file, symbol); at each level call `resolveCalls` (14) and **retain the full
   `ResolvedCall[]`** (stop classification needs external/dynamic visibility); filter
   `r.resolution.kind === "in-repo"` for scoring. Score each distinct in-repo callee
   declaration:
   `+3` same module as caller — module membership via the issue 13 partition
   (`ModuleInfo.files`); `+3` `target.exported === true` (field on the §5.2 SymbolRef
   targets from issue 14); `+2` `countBodyStatements(target) ≥ 5`; `+2` name not in
   denylist (`log debug warn error assert format parse stringify get set has`,
   case-insensitive exact); `+1` callee's own in-repo fan-out ≥ 2 (one lookahead
   resolve, cached).
2. `countBodyStatements(target, ts)`: look up the declaration by (file, startLine) via
   ts-morph; function/method/arrow → body statement count; class → constructor body
   statement count, 0 without explicit ctor; missing/abstract body → 0.
3. Next hop = highest score; ties → earliest source-order call site. Record up to
   `flow.maxFanOut − 1` other **distinct** callees with score ≥ 3 as `branchNotes`
   (format: `name (file basename)`), source order, deduped by declaration.
4. Stop conditions (set `truncated` accordingly): depth = `flow.maxDepth` (truncated
   true); no in-repo callee with score ≥ 3 (truncated false — natural end); callee
   already in the visited set (recursion; truncated false, add branchNote
   `recursion: name`); level had calls but zero in-repo resolutions (truncated true
   when ≥ 1 dynamic present at that level, else false).
5. Hop record: `{ caller, callee, callSite, branchNotes }` per §5.2, both ends §5.2
   `SymbolRef`s. The first hop's caller when `symbol` is null is the **sentinel
   SymbolRef** (no model change): `{ file: entry.file, name: "(top-level)",
   kind: "file", startLine: 1, endLine: 1, signature: "(module top-level)",
   jsdocSummary: null, exported: false }`.
6. Entries whose path has < 2 hops are dropped with warning `flow-too-shallow`
   (`entryId` in the message); the §6.1 row "no traceable entry points" applies when
   **all** entries drop (pipeline 16 maps it).
7. Caching: memoize resolveCalls results per start node within a run — key
   `file + "#top-level"` for null-symbol starts, `file + ":" + startLine + ":" + name`
   for declarations.
8. Determinism: identical inputs ⇒ identical paths (double-run test).

## Acceptance Criteria

- [ ] mini-express-app: the selected path from `src/server.ts` reaches `src/db/repo.ts` within 4 hops via routes → services (assert the hop file sequence exactly; freeze it).
- [ ] branchNotes: at the routes level, the health-route callee appears as a branch note when its score ≥ 3 (freeze actual behavior with rationale comment); a snippet with 4 significant siblings and `maxFanOut: 3` yields exactly 2 branchNotes in source order, deduped.
- [ ] Recursion snippet (a → b → a) terminates with `recursion: a` note and no infinite loop (test timeout 5 s).
- [ ] Stop-condition tests: external-only level ends `truncated: false`; a level with a dynamic call and no in-repo resolution ends `truncated: true`; equal-score tie resolves to the earlier source-order call site.
- [ ] Shallow entry (1 hop) emits no CallPath and warning `flow-too-shallow` with the entryId.
- [ ] Denylist works: a `logger.debug`-heavy snippet never selects `debug` as a hop.
- [ ] Score function: table-driven unit tests — one case per scoring rule (incl. `countBodyStatements` edge cases: no-ctor class → 0, abstract → 0), one combined case with hand-computed total.
- [ ] `maxDepth: 2` config yields exactly 2 hops with `truncated: true`.

## Validation

`pnpm --filter onboard-cli-placeholder test` (flow-select suite); reviewer reads the
frozen mini-express-app path and confirms it matches the fixture's intended §14.1 chain.

## Dependencies

13 (module partition), 14 (resolution).

## Non-goals

Multiple paths per entry (v1 = one linear path); DI-container resolution; async-boundary
special-casing (KU-5 tuning later).

## Design References

DESIGN §6.10 (selection), §5.2 CallPath, §7.3 (consumer), KU-5.
