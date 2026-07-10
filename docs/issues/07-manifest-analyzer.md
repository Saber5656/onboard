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

- `src/analyze/manifest/index.ts` — `analyzeManifest(root, files, logger) → { manifest, ciWorkflows, warnings }` (consumes FileNode[] from 06; reads only files present there).
- `src/analyze/manifest/deps-table.ts` — name→tag table (framework/server/test-runner/lint).
- Unit tests on fixtures.

## Detailed Requirements

1. `ManifestInfo` shape (add to model in this issue if not present — coordinate with §5.2):
   `{ name, version, description, packageManager: "pnpm"|"yarn"|"bun"|"npm"|null, nodeVersion: string|null, scripts: Record<string,string> (only dev/start/build/test/lint kept), bin: Record<string,string>, mainEntry: string|null (resolved main/exports target when it exists in files), workspaces: string[], depNames: string[], devDepNames: string[], detected: { frameworks: string[], testRunners: string[], linters: string[] }, otherEcosystems: { kind: "python"|"go"|"rust"|"ruby"|"jvm", file: string }[], readme: { title: string|null, firstParagraph: string|null, badgeCount: number } | null }`.
2. Malformed `package.json` → warning `manifest-parse-error`, `manifest = null`
   (contributing tour degrades per §6.1). JSON parse only — never `require()` it.
3. Lockfile precedence when several exist: pnpm-lock.yaml > yarn.lock > bun.lockb > package-lock.json (warn `manifest-multiple-lockfiles`).
4. Node version priority: `engines.node` > `.nvmrc` > `.tool-versions` (nodejs line). Store raw string.
5. Deps table (initial, extensible): frameworks `next nuxt astro remix express fastify koa @nestjs/core react vue svelte`; testRunners `vitest jest mocha ava`; linters `eslint biome prettier oxlint`. Match against dep+devDep names exactly.
6. README extraction: first `# ` heading as title; first non-heading, non-badge paragraph
   (plain text, markdown stripped naively: remove `*_`[]()` marks), ≤ 400 chars ellipsized;
   badgeCount = image links in the first 10 lines.
7. Workflows: for each `.github/workflows/*.y?(a)ml` in files: parse YAML (add `yaml`
   parser? **No new runtime dep** — implement a minimal line-based extractor: `name:`
   value, top-level `on:` keys, `jobs:` first-level key names; documented as heuristic).
   Parse failure → record `{ file, name: basename }` + warning `manifest-workflow-parse`.
8. All output arrays sorted (names asc, files asc); all strings control-char-stripped.

## Acceptance Criteria

- [ ] mini-express-app: name/scripts/engines extracted; packageManager `pnpm`; framework `express`; runner `vitest`; README title + paragraph non-null; ci.yml summarized with 2 job names.
- [ ] js-lib: `mainEntry: "index.js"`; no frameworks; otherEcosystems empty.
- [ ] plain-docs: `manifest: null` (no package.json) without warnings beyond `manifest-missing`; readme still extracted (readme lives in `docsFiles`/README parse must not depend on package.json presence — assert).
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
