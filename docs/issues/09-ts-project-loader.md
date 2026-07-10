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

1. Discovery order: `<root>/tsconfig.json`, then `<root>/src/tsconfig.json`; when
   `manifest.workspaces` is non-empty and no root tsconfig exists, still only these two
   locations (per-package projects are v2) — record `ts-workspace-shallow` warning when
   workspaces exist.
2. Load with ts-morph:
   `new Project({ tsConfigFilePath, skipAddingFilesFromTsConfig: false })`. Catch config
   parse errors → warning `ts-tsconfig-invalid`, fall through to synthesis.
3. Synthesis path (no/invalid tsconfig): `new Project({ useInMemoryFileSystem: false, compilerOptions: { allowJs: true, checkJs: false, module: NodeNext, moduleResolution: NodeNext, target: ES2022, jsx: "react-jsx", noEmit: true } })`, then `addSourceFileAtPath` for every FileNode with lang ∈ {ts, tsx, js, jsx, mts, cts, mjs, cjs} (paths from 06 — already exclusion-filtered; never re-walk the disk).
4. Guards **before** loading (evaluate on FileNode set): TS/JS-family file count > 8000
   or summed size > 128 MB → return `ts: null` + warning `ts-repo-too-large` (message
   includes both numbers and the config knobs). After loading, `project.getSourceFiles().length`
   is recorded as `sourceFileCount`.
5. Files listed in tsconfig but excluded by fs-scan (sensitive/oversize) must be removed
   from the project after load (`project.removeSourceFile`) so later stages cannot read
   them — assert in tests with a planted `.env`-adjacent ts file? (sensitive matcher is
   name-based; plant `src/secrets.yaml`-style file is not TS — instead plant an
   oversized `src/generated.ts` and assert removal).
6. Never call any API that evaluates user code (`import()`, `require`, `eval`,
   `child_process` on repo content). Lint guard: this directory gets an ESLint
   `no-restricted-imports`/`no-restricted-syntax` block for `child_process`, `eval`,
   dynamic `import(` of computed paths (added here, enforced repo-wide later by 36).
7. Return the live `Project` instance inside `TsAnalysis` (in-memory only; the model
   schema for RepoModel.ts stores derived data, not the project — coordinate with §5.2:
   `RepoModel.ts` field carries `{ sourceFileCount, tsconfigPath }` when serialized).

## Acceptance Criteria

- [ ] mini-express-app: loads via its committed tsconfig; `tsconfigPath` ends with `tsconfig.json`; sourceFileCount ≥ 10.
- [ ] js-lib: synthesized project (`tsconfigPath: null`) with all `.js` files added; `getSourceFile("index.js")` resolves.
- [ ] plain-docs (zero TS/JS files): returns `ts: null` + warning `ts-no-sources` (assert this exact code — it is distinct from `ts-repo-too-large`).
- [ ] Guard test: config-lowered cap (test-only override, e.g. 3 files) on mini-express-app → `ts: null` + `ts-repo-too-large`.
- [ ] Oversized `src/generated.ts` (planted > maxFileSizeKB) is absent from `project.getSourceFiles()` after load.
- [ ] ESLint restriction block exists and `pnpm lint` passes.

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
