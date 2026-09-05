# Title

TS project loader: tsconfig discovery, synthesized fallback, resource guards

## Summary

Implement `src/analyze/ts/` per DESIGN §6.6: locate and load the repository's TypeScript
project via ts-morph, or synthesize an in-memory default project (`allowJs`) when no
tsconfig exists, with hard file-count/byte guards that skip deep analysis on oversized
repos. Analyzed code is parsed only — never imported, never executed.

## Context

Symbols (10), imports (11), entries (12), and call paths (14–15) all run against this
loaded project. The guards are the KU-3 mitigation; the never-execute rule is threat T1.

## Scope

- `src/analyze/ts/index.ts` — `loadTsProject(root, files, config, logger) → { ts: TsAnalysis | null, warnings }` where `TsAnalysis = { project: tsMorph.Project, sourceFileCount: number, tsconfigPath: string | null }`.
- Unit tests on fixtures.

## Detailed Requirements

1. Zero TS/JS-family FileNodes → return `ts: null` + warning `ts-no-sources`
   immediately (before any loading; distinct code from the size guard).
2. Discovery order: `<root>/tsconfig.json`, then `<root>/src/tsconfig.json` — only these
   two locations in v1 (per-package projects are v2; the workspace-related warning is
   emitted by the pipeline, issue 16, which has manifest access).
3. **File admission is always explicit — ts-morph must never auto-add files.** Both
   paths construct `new Project({ compilerOptions, skipAddingFilesFromTsConfig: true })`
   and then `addSourceFileAtPath(safeJoin(root, node.path))` for exactly the FileNodes
   with lang ∈ {ts, tsx, js, jsx, mts, cts, mjs, cjs} **and** `sha256 !== ""` (fs-scan
   fully read them — sensitive/oversized/minified files are therefore never touched by
   the compiler; §6.2, §11.2 T1/T2). With a valid tsconfig, `compilerOptions` come from
   it (parse via ts-morph/TS config APIs — options only, file lists ignored);
   tsconfig parse failure → warning `ts-tsconfig-invalid`, fall through to synthesis
   defaults with `tsconfigPath: null`.
4. Synthesis defaults (no/invalid tsconfig): `{ allowJs: true, checkJs: false,
   module: NodeNext, moduleResolution: NodeNext, target: ES2022, jsx: "react-jsx",
   noEmit: true }`.
5. Guards **before** loading (evaluated on the admissible FileNode set), with an
   options parameter for tests: `loadTsProject(root, files, config, logger, options?)`
   where `options = { tsFileCap?: number (default 8000), tsByteCap?: number (default
   128 * 1024 * 1024) }` — count > cap or summed size > cap → `ts: null` + warning
   `ts-repo-too-large` (message includes both numbers and both caps). After loading,
   `project.getSourceFiles().length` is recorded as `sourceFileCount`.
6. Never call any API that evaluates user code (`import()`, `require`, `eval`,
   `child_process` on repo content). Lint guard added in the root `eslint.config.js`
   as an override for `packages/onboard/src/analyze/**`: `no-restricted-imports` for
   `child_process`/`node:child_process` (exception: `analyze/git` — its own override),
   `no-eval`, and `no-restricted-syntax` selectors `CallExpression[callee.name='eval']`
   and `NewExpression[callee.name='Function']` (repo-wide finalization in 36).
7. `TsAnalysis` carries the live `Project` (in-memory only) plus `sourceFileCount` and
   `tsconfigPath` — the latter **repo-relative POSIX** (`"tsconfig.json"` or
   `"src/tsconfig.json"`) or null; absolute paths never leave the module (§5.1, §11.9).
   The serialized `RepoModel.ts` field carries only `{ sourceFileCount, tsconfigPath }`.

## Acceptance Criteria

- [ ] mini-express-app: loads via its committed tsconfig; `tsconfigPath === "tsconfig.json"` (repo-relative); sourceFileCount ≥ 10.
- [ ] js-lib: synthesized project (`tsconfigPath: null`) with all `.js` files added; `getSourceFile` resolves `index.js`.
- [ ] plain-docs (zero TS/JS files): returns `ts: null` + warning `ts-no-sources` (assert this exact code — distinct from `ts-repo-too-large`).
- [ ] Malformed `tsconfig.json` (planted): warning `ts-tsconfig-invalid`, `tsconfigPath: null`, sources still loaded via synthesis defaults.
- [ ] Guard test: `options.tsFileCap = 3` on mini-express-app → `ts: null` + `ts-repo-too-large` with both numbers in the message.
- [ ] Exclusion proof: an oversized `src/generated.ts` (planted > maxFileSizeKB, containing a unique canary string) is never admitted — `getSourceFile` returns undefined for it **and** no loaded source text contains the canary (proves it was never read by the compiler, not merely removed after).
- [ ] ESLint restriction block exists (planted `child_process` import in `analyze/ts/` fails lint — documented spot-check) and `pnpm lint` passes.

## Validation

`pnpm --filter onboard-cli-placeholder test` (ts-loader suite); memory spot-check: loading
mini-express-app stays < 300 MB RSS delta (loose assert with `process.memoryUsage`, CI-tolerant threshold).

## Dependencies

06.

## Non-goals

Symbol/import extraction (10, 11); multi-tsconfig workspaces (v2); type-checking
diagnostics reporting.

## Design References

DESIGN §6.6, §11.2 T1, §13 (caps), ADR-005, KU-3.
