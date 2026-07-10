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

- `src/analyze/flow/select.ts` — `selectCallPaths(entryPoints, ts, files, config, logger) → { callPaths: CallPath[], warnings }`.
- `src/analyze/flow/score.ts` — significance scoring (pure function + constant table).
- Unit tests.

## Detailed Requirements

1. For each `EntryPoint` with `flowEligible !== false`: start at (file, symbol); at each
   level call `resolveCalls` (14), keep only `kind: "in-repo"` results, score each
   distinct callee declaration:
   `+3` same module as caller (module = directory-of-file prefix match against the module
   partition — pass modules in, or approximate by top-2 path segments; **use the module
   partition from issue 13**, injected as input), `+3` callee exported, `+2` callee body
   ≥ 5 statements, `+2` name not in denylist
   (`log debug warn error assert format parse stringify get set has`, case-insensitive
   exact), `+1` callee's own in-repo fan-out ≥ 2 (one lookahead resolve, cached).
2. Next hop = highest score; ties → earliest source-order call site. Record up to
   `flow.maxFanOut − 1` other distinct callees with score ≥ 3 as `branchNotes`
   (format: `name (file basename)`), source order.
3. Stop conditions (set `truncated` accordingly): depth = `flow.maxDepth` (truncated
   true); no in-repo callee with score ≥ 3 (truncated false — natural end); callee
   already in the visited set (recursion; truncated false, add branchNote
   `recursion: name`); resolver returned only external/dynamic (truncated true when any
   dynamic present, else false).
4. Hop record: `{ caller: SymbolRef-like of current, callee: target, callSite, branchNotes }`
   per §5.2. The first hop's caller is the entry symbol (or synthetic
   `{ name: "(top-level)", kind: "file" }` when symbol is null — extend the model
   union accordingly, coordinate with §5.2).
5. Entries whose path has < 2 hops are dropped with warning `flow-too-shallow` (the
   entry-flow tour for them is skipped; §7.3).
6. Caching: memoize resolveCalls per declaration within a run (pure map key =
   file + startLine).
7. Determinism: identical inputs ⇒ identical paths (double-run test).

## Acceptance Criteria

- [ ] mini-express-app: the selected path from `src/server.ts` reaches `db/repo.ts` within 4 hops via routes → services (assert the hop file sequence exactly; freeze it).
- [ ] branchNotes: at the routes level, the health-route callee appears as a branch note when its score ≥ 3 (freeze actual behavior with rationale comment).
- [ ] Recursion snippet (a → b → a) terminates with `recursion: a` note and no infinite loop (test timeout 5 s).
- [ ] Denylist works: a `logger.debug`-heavy snippet never selects `debug` as a hop.
- [ ] Score function: table-driven unit tests — one case per scoring rule, one combined case with hand-computed total.
- [ ] `maxDepth: 2` config yields exactly 2 hops with `truncated: true`.

## Validation

`pnpm --filter onboard-cli-placeholder test` (flow-select suite); reviewer reads the
frozen mini-express-app path and confirms it matches the fixture's intended §14.1 chain.

## Dependencies

14 (+ modules from 13 as scoring input).

## Non-goals

Multiple paths per entry (v1 = one linear path); DI-container resolution; async-boundary
special-casing (KU-5 tuning later).

## Design References

DESIGN §6.10 (selection), §5.2 CallPath, §7.3 (consumer), KU-5.
