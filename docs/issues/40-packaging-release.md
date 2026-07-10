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

- Name resolution: check availability of candidates at implementation time (`npm view <name>` for e.g. `onboard-tour`, `onboardkit`, scoped `@<owner-org>/onboard`); present the verified list to the owner as an issue comment; apply the decision repo-wide (package name, README placeholder (39), schema `$schema` URL, ISSUE_PLAN note).
- `packages/onboard/package.json` finalization: `files: ["dist", "assets/viewer", "schemas", "README.md", "LICENSE"]`, `exports` (root: lib entry; `./package.json`), `bin`, `engines`, repository/bugs/homepage metadata, `publishConfig: { access: "public", provenance: true }`, exact pins for shiki/markdown-it/ts-morph (ADR-007 — verify still pinned), license field final (owner confirms MIT or alternative — ask in the PR if unconfirmed).
- `prepack`: builds viewer + CLI and verifies `assets/viewer/app.js` exists (fail loud).
- `.github/workflows/release.yml`: on tag `v*` — full CI gate reuse, `pnpm publish` with provenance, GitHub Release with generated notes; requires repo secret `NPM_TOKEN` (documented owner setup steps; workflow fails with a clear message when absent); `workflow_dispatch` dry-run mode (`pnpm publish --dry-run`) for rehearsal.
- Pack verification test: `pnpm pack` in CI (regular pipeline, not just release) + assert tarball contents match the whitelist exactly (no fixtures, no tests, no `.onboard`).

## Detailed Requirements

1. Tarball budget: warn > 15 MB (Shiki themes/langs dominate; document actual size).
2. `npx <name>@latest --help` path proven via the dry-run rehearsal on a local registry
   (verdaccio in a test) **or** documented as release-checklist step 1 post-publish —
   choose the verdaccio test (offline, repeatable) and implement it.
3. Version starts `0.1.0`; release checklist in `docs/release.md`: gates green → tag →
   workflow → post-publish smoke (`npx <name> generate` on a scratch repo) → GitHub
   Release notes review. Merge-to-main ≠ release (user rule) — the tag is the explicit
   release act, owner-performed.
4. Release workflow permissions: `contents: write` (release notes) + `id-token: write`
   (provenance) only; no other scopes.
5. The `$schema` URL in generated configs (03) switches from the placeholder to the
   published package path (`node_modules/<name>/schemas/...`) — regression test updated.

## Acceptance Criteria

- [ ] Owner has confirmed the package name from the verified-available list (recorded in this issue's PR/GitHub issue thread) and it is applied everywhere (repo-wide grep for the placeholder returns nothing).
- [ ] `pnpm pack` tarball contains exactly the whitelist (test-asserted) and installs+runs (`onboard --version`) in the verdaccio test.
- [ ] Release workflow dry-run (workflow_dispatch) succeeds end-to-end on CI without publishing.
- [ ] Provenance + access flags present; `NPM_TOKEN` absence produces the documented clear failure.
- [ ] `docs/release.md` checklist complete; license confirmed by owner.

## Validation

CI pack/verdaccio tests green; dry-run workflow log linked in PR; owner sign-offs
(name, license) recorded.

## Dependencies

38 (CI gates), 39 (README placeholder), 01 (packaging skeleton).

## Non-goals

The actual first publish (owner-gated); npm org creation; changelog automation
(release notes generated is enough for v1); Homebrew/binary distribution.

## Design References

DESIGN §11.8, §3, ADR-001, ADR-007, research §4, KU-1.
