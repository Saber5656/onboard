# Title

Manifest analyzer: package.json, README, CI workflows, ecosystem detection

## Summary

Implement `src/analyze/manifest/` per DESIGN §6.3: parse root `package.json` (+ lockfile
type, engines, workspaces, scripts, dependency names), detect frameworks/test runners
from a dependency table, shallow-detect other ecosystems (pyproject/go.mod/Cargo.toml/
Gemfile/pom.xml), extract README head, and summarize `.github/workflows/*.yml`.

## Context

Feeds entry-point evidence (§6.9), the contributing tour (§7.5), and narration facts
("uses express, vitest"). Must degrade gracefully: none of these files are guaranteed.

## Scope

- `src/analyze/manifest/index.ts` — `analyzeManifest(root, files, logger) → { manifest, readme, ciWorkflows, warnings }` (consumes FileNode[] from 06; reads only files present there, always via `safeJoin(root, rel)` — §11.3).
- `src/analyze/manifest/deps-table.ts` — name→tag table (framework/server/test-runner/lint).
- Unit tests on fixtures.

## Detailed Requirements

1. `ManifestInfo` shape (finalizes the issue-02 placeholder; matches §5.2):
   `{ name, version, description, packageManager: "pnpm"|"yarn"|"bun"|"npm"|null, nodeVersion: string|null, scripts: Record<string,string> (only dev/start/build/test/lint kept), bin: Record<string,string>, mainEntry: string|null (resolution rule 6), workspaces: string[], depNames: string[], devDepNames: string[], detected: { frameworks: string[], testRunners: string[], linters: string[] }, otherEcosystems: { kind: "python"|"go"|"rust"|"ruby"|"jvm", file: string }[] }`.
   README data is **not** part of ManifestInfo: it is returned separately and stored on
   `RepoModel.readme` (§5.2) so repos without package.json keep it. CONTRIBUTING/LICENSE/
   CODE_OF_CONDUCT presence is already provided by fs-scan `docsFiles` (§6.2) — do not
   re-detect here.
2. Content extraction only reads files that fs-scan fully read (`sha256 !== ""`);
   lockfile and config detection that needs only *presence* uses the FileNode path set.
   Malformed `package.json` → warning `manifest-parse-error`, `manifest = null`
   (contributing tour degrades per §6.1). JSON parse only — never `require()` it.
3. Lockfile precedence when several exist: pnpm-lock.yaml > yarn.lock > bun.lockb > package-lock.json (warn `manifest-multiple-lockfiles`).
4. Node version priority: `engines.node` > `.nvmrc` > `.tool-versions` (nodejs line). Store raw string.
5. Deps table exactly the DESIGN §6.3 list: frameworks `next nuxt astro vite express
   fastify koa react vue svelte` plus mapping `@nestjs/core → nest`; testRunners
   `vitest jest mocha` plus rule "scripts.test contains `node --test` → node:test";
   linters `eslint biome prettier`. Match against dep+devDep names exactly; extending
   the table is a DESIGN §6.3 change first.
6. `mainEntry` resolution (deterministic v1 heuristic): try in order `main`,
   `exports` when it is a string, `exports["."]` when string, `exports["."].default`
   then `exports["."].import` when strings — accept only a relative path that exists in
   the FileNode set (else null). Arrays and other conditional forms → null.
7. README extraction (→ `readme` output): first `# ` heading as title; first
   non-heading, non-badge paragraph (plain text, markdown stripped naively: remove
   ``*_`[]()`` marks), ≤ 400 chars ellipsized; badgeCount = image links in the first
   10 lines. Runs whenever fs-scan found a readme docsFile, independent of package.json.
8. Workflows: for each `.github/workflows/*.yml` or `*.yaml` in files: minimal
   line-based extraction (no YAML dependency): `name:` value; `on:` triggers supporting
   the three forms — scalar (`on: push`), list (`on: [push, pull_request]`), and map
   (indented keys under `on:`); `jobs:` first-level key names. Documented as heuristic;
   failure → record `{ file, name: basename }` + warning `manifest-workflow-parse`.
9. All output arrays sorted (names asc, files asc); all strings control-char-stripped.

## Acceptance Criteria

- [ ] mini-express-app: name/scripts/engines extracted; packageManager `pnpm`; framework `express`; runner `vitest`; readme title + paragraph non-null; ci.yml summarized with its workflow `name`, `on:` trigger keys, and 2 job names.
- [ ] Workflow trigger forms: scalar, list, and map `on:` variants (three inline fixtures) each extract the expected trigger names.
- [ ] js-lib: `mainEntry: "index.js"`; no frameworks; otherEcosystems empty.
- [ ] `exports`-only package fixtures: string form, `"."`-string form, and `"."`-conditional (`default`/`import`) form each resolve; an array-form `exports` yields `mainEntry: null`.
- [ ] plain-docs: `manifest: null` with warning `manifest-missing`; `readme` still extracted (asserts independence from package.json).
- [ ] Malformed package.json fixture (inline in test) → `manifest: null` + `manifest-parse-error`, no throw.
- [ ] A `pyproject.toml` planted in a temp fixture is reported as `{ kind: "python" }`.

## Validation

`pnpm --filter onboard-cli-placeholder test` (manifest suite). Reviewer checks the
ManifestInfo fields against §6.3 line by line.

## Dependencies

06.

## Non-goals

Version-range analysis of deps; executing scripts; full YAML fidelity; per-workspace
manifests (v2 monorepo work).

## Design References

DESIGN §6.3, §6.1 (degradation), §7.5 (consumer), §5.2.
