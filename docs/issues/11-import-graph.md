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

- `src/analyze/imports/index.ts` — `buildImportGraph(ts: TsAnalysis | null, files, logger) → { edges: { from: string, to: string }[], externals: { pkg: string, importCount: number }[], warnings }`; `ts: null` → empty edges/externals, no warning (the loader already warned).
- Unit tests on fixtures.

## Detailed Requirements

1. Specifier sources per file: `ImportDeclaration`, `ExportDeclaration` with module
   specifier (`export … from`, `export * from`), `require("literal")` call expressions,
   `import("literal")` dynamic imports with string-literal argument (edge extraction for
   the literal call forms is normative per DESIGN §6.8). Non-literal dynamic
   import/require → increment per-file counter, single warning
   `imports-dynamic-unresolved` (file + count).
2. Resolution — two paths: (a) import/export declarations use ts-morph's
   `getModuleSpecifierSourceFile()`; (b) literal `require(...)`/`import(...)` call
   expressions use the TypeScript module-resolution API with the containing file and the
   project's compiler options. A resolved file counts as in-repo only when its
   normalized repo-relative POSIX path is present in the fs-scan FileNode set; edge
   endpoints are always those normalized paths (never absolute, §5.1).
3. Classification of non-in-repo resolutions:
   - bare specifiers → external package name by exact algorithm: `node:*` **and** Node
     builtin names → `node`; `@scope/name[/…]` → `@scope/name`; `name[/…]` → `name`;
   - relative specifiers with an asset extension
     (css scss sass less styl svg png jpg jpeg gif webp ico woff woff2) → silently
     ignored (no edge, no external, no warning);
   - other relative specifiers that fail to resolve → warning `imports-unresolved`
     (file + specifier), no edge;
   - relative specifiers that resolve to a file **not** in the FileNode set
     (excluded/oversized/sensitive) → warning `imports-out-of-scope`, no edge, no
     external count (§11 boundary — never surface excluded paths beyond the warning).
4. `externals[].importCount` = number of literal import/export/require/import() sites
   referencing the package, counted before edge dedupe (per occurrence, per file).
5. Output: `edges` deduped, self-edges dropped, sorted by (from, to) with the shared
   `byteOrderCompare` (§5.6). `externals` sorted by importCount desc then pkg asc.
6. Warnings use the `PipelineWarning` shape (§6.1): `stage: "imports"`, stable `code`,
   message without stacks, sanitized repo-relative `path`. Never throw for content
   reasons; per-file try/catch → `imports-file-error` warning.

## Acceptance Criteria

- [ ] mini-express-app: edges include `src/server.ts → src/routes/users.ts`, `src/routes/users.ts → src/services/userService.ts`, `src/services/userService.ts → src/db/repo.ts`; externals include `express` with count ≥ 1; `node` builtin grouping asserted.
- [ ] `export { x } from "./x"` and `export * from "./y"` snippets each produce the expected edge.
- [ ] No duplicate edges when a file imports the same target twice (type + value import); `importCount` still counts both occurrences.
- [ ] A planted `import(dynamicVar)` yields `imports-dynamic-unresolved` and no edge; a planted `import "./style.css"` yields nothing (no edge/external/warning); a planted `import "./missing"` yields `imports-unresolved`; a per-file throw seam yields `imports-file-error` with `stage: "imports"` and a sanitized path.
- [ ] js-lib with `require("./lib/parse")` produces `index.js → lib/parse.js`; a literal `import("./lib/format.js")` produces its edge.
- [ ] Deep imports `express/lib/router` and `@scope/pkg/sub` (planted snippets) count as `express` and `@scope/pkg`.
- [ ] `ts: null` input returns empty results without warnings.

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
