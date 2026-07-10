# Title

Scaffold pnpm workspace, toolchain, and CI skeleton

## Summary

Create the monorepo skeleton exactly as DESIGN §2.4: pnpm workspaces with
`packages/onboard` (published CLI+library) and `packages/viewer` (private Preact SPA),
shared TypeScript/ESLint/Prettier/vitest toolchain, viewer→CLI asset build wiring, and a
minimal CI workflow (lint, typecheck, test) on Node 22 and 24.

## Context

Nothing exists yet except `README.md`. Every later issue assumes this layout, these
package names, and these scripts. Getting paths and scripts exactly right here removes
guesswork from all 40 downstream issues.

## Scope

- Root: `package.json` (private, `"type": "module"`, `engines.node: ">=20"`), `pnpm-workspace.yaml` (`packages/*`), `.gitignore`, `.editorconfig`, base `tsconfig.base.json`.
- `packages/onboard`: `package.json` (temporary name `onboard-cli-placeholder` — final name decided in issue 40/KU-1; `bin: { "onboard": "./dist/cli/main.js" }`), `tsconfig.json`, placeholder `src/cli/main.ts` printing version, `src/index.ts` empty export.
- `packages/viewer`: `package.json` (private), Vite config building a **single IIFE bundle** to `../onboard/assets/viewer/` (`app.js`, `app.css`, `index.html` template unused at runtime — emitter composes HTML), placeholder `src/app.tsx` rendering "onboard viewer placeholder".
- Toolchain: eslint 9 flat config + typescript-eslint, prettier 3, vitest 4 config in both packages (one passing placeholder test each).
- CI: `.github/workflows/ci.yml` — pnpm install (frozen lockfile), lint, typecheck, `pnpm -r test`, `pnpm -r build`; matrix Node 22.x / 24.x, ubuntu-latest.
- License file: MIT (placeholder; final license confirmed in issue 40).

## Detailed Requirements

1. Dependency versions per DESIGN §3 (majors as of 2026-07): typescript `^7.0` (devDep; if `tsc` v7 breaks eslint/vitest integration, fall back to `~5.9` and record the fallback in the PR description — KU-2), vitest `^4`, eslint `^9`, prettier `^3`, preact `^10`, vite `^8`, tsx `^4` (dev script only). No runtime deps yet beyond commander/zod placeholders — do **not** add any dependency not listed in DESIGN §3.
2. `pnpm-lock.yaml` is generated and **committed**; CI activates pnpm via corepack (`packageManager` field) and installs with `pnpm install --frozen-lockfile`.
3. Scripts (root): `build` (viewer build first, then `tsc -b` for onboard — ordered, not `pnpm -r build`), `test` (`pnpm -r test`), `lint`, `typecheck`, `format`. Scripts (onboard): `build`, `test`, `dev` (`tsx src/cli/main.ts`). Scripts (viewer): `build`, `test`. CI runs the root `build` script so ordering is deterministic.
4. Viewer Vite config — exact settings (default app builds do not emit these filenames):
   `build.rollupOptions.input = "src/main.ts"` (or the entry html-less equivalent),
   `build.rollupOptions.output = { format: "iife", entryFileNames: "app.js", assetFileNames: "app.css" }`,
   `build.rollupOptions.output.inlineDynamicImports = true` (single chunk, no code splitting),
   `build.cssCodeSplit = false`, `outDir: "../onboard/assets/viewer"`, `emptyOutDir: true`.
   Acceptance asserts the exact output filenames.
5. `.gitignore`: `node_modules/`, `dist/`, `packages/onboard/assets/viewer/`, `.onboard/`, `coverage/` (the lockfile is NOT ignored).
6. ESLint: enable an empty `no-restricted-syntax`/`no-restricted-imports` skeleton with comments naming the issues that fill it (09: no child_process/eval in analyzers; 29: `dangerouslySetInnerHTML` only in audited sinks; 36: repo-wide finalization). Keep default recommended sets otherwise.
7. pnpm version pinned via `packageManager` field (current pnpm line at implementation time).
8. Placeholder CLI output format: `main.ts` reads its own package.json `version` and prints exactly `onboard <version>` (e.g. `onboard 0.0.0`) to stdout, exit 0.
9. Directory creation scope: this issue creates only the paths listed in Scope. The full DESIGN §2.4 tree is built up by later issues — do not pre-create empty directories.

## Acceptance Criteria

- [ ] `pnpm install --frozen-lockfile && pnpm build && pnpm test && pnpm lint && pnpm typecheck` all succeed from a clean clone; `git ls-files pnpm-lock.yaml` shows the lockfile is committed.
- [ ] `node packages/onboard/dist/cli/main.js` prints exactly `onboard <version>` (version from package.json) and exits 0.
- [ ] `packages/onboard/assets/viewer/app.js` and `app.css` exist after build with those exact names; `app.js` contains an IIFE (assert no top-level `import ` statement).
- [ ] CI workflow runs the same commands (with `--frozen-lockfile`) on Node 22 and 24.
- [ ] Every path this issue creates matches its DESIGN §2.4 location.

## Validation

Run the Acceptance commands locally; open a draft PR and confirm CI is green on both Node
versions. `git ls-files` shows no `dist/`, `assets/viewer/`, or lockfile-external files
beyond those specified.

## Dependencies

None (first issue).

## Non-goals

Any analyzer/tour/server code; publishing configuration (issue 40); complete CI gates
(issue 38); final package name (KU-1).

## Design References

DESIGN §2.4 (layout), §3 (stack table), §14.6 (CI), ADR-001, ADR-004; ISSUE_PLAN wave 0.
