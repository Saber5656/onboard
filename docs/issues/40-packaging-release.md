# Title

npm packaging, package-name decision, provenance release workflow

## Summary

Make the package publishable per DESIGN §11.8 and ADR-001: resolve KU-1 (final npm
package name — owner decision from verified-available candidates), finalize
`packages/onboard/package.json` (files whitelist, exports, bin, exact pins, metadata),
add the tag-triggered release workflow with `npm publish --provenance`, and the release
checklist. **Publishing itself is a manual, owner-gated action** — this issue delivers
the mechanism, not the act.

## Context

`onboard` is taken on npm (research §4). Name choice affects README (39 placeholder),
the `$schema` path (03), and nothing else by design (bin stays `onboard`). npm token
setup is the owner's manual step per the user's security rules (agents never handle
credentials).

## Scope

- Name resolution: check availability of candidates at implementation time (`npm view <name>` for e.g. `onboard-tour`, `onboardkit`, scoped `@<owner-org>/onboard` — note `npm view` only proves the name is currently unclaimed; scoped candidates additionally require the owner to confirm they control the npm org/scope); present the verified list to the owner as an issue comment; apply the decision repo-wide (package name, README placeholder (39), schema `$schema` URL, ISSUE_PLAN note).
- `packages/onboard/package.json` finalization: `files: ["dist", "assets/viewer", "schemas", "README.md", "LICENSE"]` (paths relative to `packages/onboard` — `prepack` **copies the root README.md and LICENSE into the package dir**, gitignored there), `exports` (root: lib entry; `./package.json`), `bin`, `engines`, repository/bugs/homepage metadata, `publishConfig: { access: "public", provenance: true }`, exact pins for shiki/markdown-it/ts-morph (ADR-007 — verify still pinned), license field final (owner confirms MIT or alternative — ask in the PR if unconfirmed).
- `prepack`: builds viewer + CLI, stages README/LICENSE, and verifies `assets/viewer/app.js` exists (fail loud).
- `.github/workflows/release.yml`: **`workflow_dispatch` only** (inputs: `tag` to release, `dryRun` boolean) — no automatic publish on tag push; the publish job runs in a protected GitHub Environment `release` requiring the owner's manual approval (documented owner setup steps). Gates: invokes the CI workflow via its `workflow_call` trigger (38) and publishes only when all gates pass. `dryRun: true` → `pnpm publish --dry-run`, requires **no** `NPM_TOKEN`; real publish fails fast with a clear message when `NPM_TOKEN` is absent. GitHub Release with generated notes after successful publish.
- Pack verification test: `pnpm pack` in CI (regular pipeline, not just release) + assert the tarball file list equals exactly `package/package.json` + the whitelisted files/dirs (npm always includes package.json; paths are under `package/`) — no fixtures, no tests, no `.onboard`, no source maps.

## Detailed Requirements

1. Tarball budget: warn > 15 MB (Shiki themes/langs dominate; document actual size).
2. Verdaccio test (offline, repeatable), exact contract: devDep `verdaccio` started
   in-process on an ephemeral 127.0.0.1 port with a temp storage dir; temp `HOME` +
   `.npmrc` pointing registry at it (never the user's real config); `pnpm publish
   --registry <local> --no-git-checks` of the packed package; then in a scratch dir
   `npm install --registry <local> <name>@0.1.0` and assert
   `node_modules/.bin/onboard --version` exits 0 printing the version — no request
   leaves localhost (assert via registry logs).
3. Version starts `0.1.0`; release checklist in `docs/release.md`: gates green → owner
   runs the dispatch workflow (dry-run first, then real with environment approval) →
   post-publish smoke (`npx <name> generate` on a scratch repo) → GitHub Release notes
   review. Merge-to-main ≠ release (owner rule) — the dispatch + environment approval
   is the explicit release act, owner-performed.
4. Release workflow permissions: `contents: write` (release notes) + `id-token: write`
   (provenance) only; no other scopes.
5. The `$schema` URL in generated configs (03) switches from the placeholder to the
   exact published path `./node_modules/<name>/schemas/onboard.config.schema.json`
   (§4.3 string) — the 03 regression test asserts this exact value.

## Acceptance Criteria

- [ ] Owner has confirmed the package name from the verified-available list (scope-access confirmed for scoped names; recorded in this issue's thread) and it is applied everywhere (repo-wide grep for the placeholder returns nothing).
- [ ] `pnpm pack` tarball file list equals exactly `package/package.json` + whitelist (test-asserted) and installs+runs (`onboard --version`) in the verdaccio test without leaving localhost.
- [ ] Release workflow dry-run (workflow_dispatch, `dryRun: true`) succeeds end-to-end on CI **without** `NPM_TOKEN` and without publishing; the real-publish path fails fast with the documented message when `NPM_TOKEN` is absent.
- [ ] The publish job is bound to the protected `release` environment (owner-approval gate) — verified in the workflow file and documented in `docs/release.md`.
- [ ] Provenance + access flags present; `docs/release.md` checklist complete; license confirmed by owner.

## Validation

CI pack/verdaccio tests green; dry-run workflow log linked in PR; owner sign-offs
(name, license) recorded.

## Dependencies

38 (CI gates + workflow_call), 39 (README placeholder).

## Non-goals

The actual first publish (owner-gated); npm org creation; changelog automation
(release notes generated is enough for v1); Homebrew/binary distribution.

## Design References

DESIGN §11.8, §3, ADR-001, ADR-007, research §4, KU-1.
