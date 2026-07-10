# Title

Call-path tracer (1/2): call resolution engine

## Summary

Implement the resolution half of DESIGN §6.10 in `src/analyze/flow/resolve.ts`: given a
starting declaration (entry symbol or file top-level), enumerate its call expressions in
source order and resolve each callee to an **in-repo** declaration via the ts-morph type
checker, classifying every call site as resolved / external / unresolvable-dynamic.

## Context

This is the technically hardest v1 module (KU-5). Splitting resolution (this issue) from
path selection (15) lets each be tested mechanically. Resolution correctness bounds the
quality of every entry-flow tour.

## Scope

- `src/analyze/flow/resolve.ts` — `resolveCalls(start: StartNode, ts: TsAnalysis, files) → ResolvedCall[]` where
  `StartNode = { file: string, symbol: string | null }` (`symbol` = a declaration name
  in that file; lookup = first declaration with that name by startLine; not found →
  empty result + warning `flow-start-missing`) and
  `ResolvedCall = { callSite: Anchor, calleeName: string, resolution: { kind: "in-repo", target: SymbolRef } | { kind: "external", pkg: string | null } | { kind: "dynamic" } }`.
  Targets use the §5.2 `SymbolRef` shape — non-exported declarations are allowed with
  `exported: false`, `jsdocSummary` populated when present; `file` is repo-relative
  POSIX and `name`/`signature`/`jsdocSummary` pass the same formatting + 
  `stripControlChars` rules as issue 10 (§11.3; reuse `signature.ts`).
- Unit tests on fixtures + focused snippets.

## Detailed Requirements

1. Start body: when `symbol` given → that declaration's body; when null → the file's
   top-level statements (in order). Missing body (declaration file, abstract) →
   empty result.
2. Enumerate `CallExpression` and `NewExpression` nodes in **source order**, including
   inside nested callbacks/arrow functions of the start body, but not inside nested
   function *declarations* (those are separate potential hops).
3. Resolve callee, handling exactly these shapes (anything else → `dynamic`):
   - identifier call `foo()` → declaration via type-checker symbol;
   - property access `obj.method()`: receiver typed as a project **class** → that
     class's method declaration; receiver typed as a project **interface** → collect
     classes in FileNode files with an explicit `implements ThatInterface` clause that
     declare the method — exactly one match resolves to it, zero or > 1 → `dynamic`;
     ambient/external receiver types → external/dynamic per rule 4;
   - static method `Class.create()` → the static method declaration;
   - `new Class()` → constructor declaration (or class when no explicit ctor);
   - namespace import call `ns.fn()` → the exported function in the resolved module.
4. In-repo test: resolved declaration's source file path (normalized repo-relative
   POSIX) is in the FileNode set → in-repo. Otherwise `external` with `pkg` by the
   same extraction algorithm as issue 11 rule 3 (`node:*`/builtins → `node`,
   `@scope/name[/…]` → `@scope/name`, `name[/…]` → `name`); ambient/global targets
   (`console.log`) and project-file targets **outside** the FileNode set (excluded/
   oversized) → `pkg: null` (the excluded path never appears in output).
5. Each `ResolvedCall.callSite` anchor = call expression's line span in the **caller**
   file (1-based, clamped to the file).
6. Robustness: per-node try/catch → classify `dynamic`; never throw for content.
   Recursion guard is issue 15's concern (visited set lives in selection).
7. Determinism: output order = source order; no randomness; type-checker results must be
   accessed through a single helper for testability.

## Acceptance Criteria

- [ ] mini-express-app, from `src/server.ts` top-level, in source order (frozen expectations per call): `express()` → external `express`; `app.use(express.json())` → external; `registerUserRoutes(app)` → **in-repo** to `src/routes/users.ts` (the fixture's §14.1 direct call — this is the hop the flow tour rides); `app.listen(...)` → external `express` or `dynamic` (freeze actual with comment).
- [ ] From the users route handler: `userService.createUser()`-style property call resolves in-repo to `src/services/userService.ts` declaration with correct line span.
- [ ] Snippet tests (inline project): identifier call, static method, `new` ctor, namespace import call each resolve in-repo; interface with two `implements` classes → `dynamic`; interface with exactly one → in-repo; `arr.map(fn)` → external/dynamic (frozen), never in-repo.
- [ ] Traversal boundary snippet: a top-level call, a call inside an inline arrow callback (both enumerated, in order), and a call inside a nested function declaration (not enumerated) — exact expected sequence asserted.
- [ ] Start-node behavior: symbol start (body calls only), null start (top-level statements), missing symbol → empty + `flow-start-missing`, declaration-only body → empty.
- [ ] `console.log` → external with `pkg: null`.
- [ ] Two runs produce identical serialized ResolvedCall arrays for the fixture.

## Validation

`pnpm --filter onboard-cli-placeholder test` (flow-resolve suite). Reviewer reads the
frozen classification snapshot for mini-express-app and confirms each entry is
defensible.

## Dependencies

10 (SymbolRef shape reuse), 12 (StartNode inputs).

## Non-goals

Choosing which call becomes the next hop (15); cross-hop traversal (15); framework
middleware-chain modeling (KU-5, v2 tuning).

## Design References

DESIGN §6.10 (resolution rules), §5.2 CallPath/Anchor, KU-5.
