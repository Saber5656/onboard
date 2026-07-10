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

1. Dependency versions per DESIGN §3 (majors as of 2026-07): typescript `^7.0` (devDep; if `tsc` v7 breaks eslint/vitest integration, fall back to `~5.9` and record the fallback in the PR description — KU-2), vitest `^4`, eslint `^9`, prettier `^3`, preact `^10`, vite `^8`. No runtime deps yet beyond commander/zod placeholders — do **not** add any dependency not listed in DESIGN §3.
2. Scripts (root): `build` (viewer build then `tsc -b` for onboard), `test` (`pnpm -r test`), `lint`, `typecheck`, `format`. Scripts (onboard): `build`, `test`, `dev` (`tsx src/cli/main.ts` or `node --watch` equivalent). Scripts (viewer): `build`, `test`.
3. Viewer Vite config: `build.rollupOptions.output.format = "iife"`, single chunk (no code splitting), `outDir: "../onboard/assets/viewer"`, `emptyOutDir: true`, no hashed filenames (`app.js`, `app.css` exact — the emitter references them literally).
4. `.gitignore`: `node_modules/`, `dist/`, `packages/onboard/assets/viewer/`, `.onboard/`, `coverage/`.
5. ESLint: enable `no-restricted-syntax` skeleton (rules filled by later issues), forbid `any` escalation defaults off (keep default recommended sets; strictness increases later, not here).
6. pnpm version pinned via `packageManager` field (current pnpm 9/10 line at implementation time).

## Acceptance Criteria

- [ ] `pnpm install && pnpm build && pnpm test && pnpm lint && pnpm typecheck` all succeed from a clean clone.
- [ ] `node packages/onboard/dist/cli/main.js` prints a version string and exits 0.
- [ ] `packages/onboard/assets/viewer/app.js` exists after build and contains an IIFE (starts with `(function` or `!function` or `(() =>` — assert no `import ` statement at top level).
- [ ] CI workflow runs the same five commands on Node 22 and 24.
- [ ] Repository layout matches DESIGN §2.4 for every directory this issue creates.

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
