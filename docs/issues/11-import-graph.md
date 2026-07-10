# Title

Import graph builder (file-level, resolved)

## Summary

Implement `src/analyze/imports/` per DESIGN §6.8: for every source file in the ts-morph
project, resolve static import/export-from/require specifiers to in-repo target files,
emit a deduplicated sorted edge list plus external-package usage counts, and count
unresolvable dynamic imports.

## Context

Edges drive module fanIn/fanOut (13), module ranking in the architecture tour (18), and
the dep-graph widget layout. External package counts feed narration ("uses express, zod").

## Scope

- `src/analyze/imports/index.ts` — `buildImportGraph(ts: TsAnalysis, files, logger) → { edges: { from: string, to: string }[], externals: { pkg: string, importCount: number }[], warnings }`.
- Unit tests on fixtures.

## Detailed Requirements

1. Specifier sources per file: `ImportDeclaration`, `ExportDeclaration` with module
   specifier, `require("literal")` call expressions, `import("literal")` dynamic imports
   with string-literal argument. Non-literal dynamic import/require → increment
   per-file counter, single warning `imports-dynamic-unresolved` (file + count).
2. Resolution: use the module specifier's resolved source file from ts-morph
   (`getModuleSpecifierSourceFile()`); fallback for `.js`-suffixed NodeNext specifiers
   already handled by the compiler. Resolved file must be inside the project **and**
   present in the fs-scan FileNode set; otherwise treat as external.
3. External classification: bare specifiers map to package name (`@scope/pkg` keeps
   scope; deep imports `pkg/sub` count for `pkg`; `node:*` builtins grouped as `node`).
   Relative specifiers that fail to resolve → warning `imports-unresolved` (file +
   specifier), not an edge.
4. Output: `edges` deduped, self-edges dropped, sorted by (from, to) byte order.
   `externals` sorted by importCount desc then pkg asc.
5. Never throw for content reasons; per-file try/catch → `imports-file-error` warning.

## Acceptance Criteria

- [ ] mini-express-app: edges include `src/server.ts → src/routes/users.ts`, `src/routes/users.ts → src/services/userService.ts`, `src/services/userService.ts → src/db/repo.ts`; externals include `express` with count ≥ 1; `node` builtin grouping asserted.
- [ ] No duplicate edges when a file imports the same target twice (type + value import).
- [ ] A planted `import(dynamicVar)` yields `imports-dynamic-unresolved` and no edge.
- [ ] js-lib with `require("./lib/parse")` produces `index.js → lib/parse.js`.
- [ ] Deep import `express/lib/router` (planted snippet) counts as `express`.

## Validation

`pnpm --filter onboard-cli-placeholder test` (imports suite); determinism double-run
equality on serialized output.

## Dependencies

09.

## Non-goals

Module-level aggregation (13); cycle analysis (18 handles layout-time cycle breaking);
CSS/asset imports (ignored — non-source langs resolve to nothing).

## Design References

DESIGN §6.8, §6.5 (aggregation consumer), §5.5 dep-graph widget.
