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
  `StartNode = { file, symbol | null }` and
  `ResolvedCall = { callSite: Anchor, calleeName: string, resolution: { kind: "in-repo", target: SymbolLikeRef } | { kind: "external", pkg: string | null } | { kind: "dynamic" } }`;
  `SymbolLikeRef = { file, name, kind, startLine, endLine, signature }` (superset target: includes non-exported declarations — flow may traverse them even though issue 10 indexes only exports).
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
   - property access `obj.method()` where `obj`'s type is a project class/interface with
     exactly one project implementation → that method declaration; > 1 implementation
     or ambient type → `dynamic`;
   - static method `Class.create()` → the static method declaration;
   - `new Class()` → constructor declaration (or class when no explicit ctor);
   - namespace import call `ns.fn()` → the exported function in the resolved module.
4. In-repo test: resolved declaration's source file is in the project **and** its path is
   in the FileNode set; else `external` with package name extracted from the module
   specifier when determinable (`null` for ambient/global like `console.log`).
5. Each `ResolvedCall.callSite` anchor = call expression's line span in the **caller**
   file (1-based, clamped to the file).
6. Robustness: per-node try/catch → classify `dynamic`; never throw for content.
   Recursion guard is issue 15's concern (visited set lives in selection).
7. Determinism: output order = source order; no randomness; type-checker results must be
   accessed through a single helper for testability.

## Acceptance Criteria

- [ ] mini-express-app: resolving from `src/server.ts` (top-level) yields, in order, in-repo resolutions covering the route-mount call into `routes/users.ts` and app-setup calls; `app.listen(...)` resolves `external` (express) or `dynamic` — freeze actual classification with a comment.
- [ ] From the users route handler: `userService.createUser()`-style property call resolves in-repo to `services/userService.ts` declaration with correct line span.
- [ ] Snippet tests (inline project): identifier call, static method, `new` ctor, namespace import call each resolve in-repo; interface with two implementations resolves `dynamic`; `arr.map(fn)` resolves `external`/`dynamic` (frozen), never in-repo.
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
