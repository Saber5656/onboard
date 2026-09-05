# Title

Determinism gate, golden bundle, self-host E2E, viewer smoke

## Summary

Implement the §14.3–14.4 CI-blocking gates: the double-run byte-equality determinism
test, the committed golden bundle with an intentional-update script, the self-hosting
E2E (`onboard generate` on the onboard repo itself), and the Playwright viewer smoke
(static site + serve mode with mock chat).

## Context

These gates convert the two product claims (§1.2) into enforced properties: determinism
is proven by hashes, and the end-to-end path is proven by driving the real viewer.

## Scope

- `test/e2e/determinism.test.ts` — generate twice on materialized mini-express-app → compare **sorted manifests** of `{ relativePath, sha256, size }` covering `tour-bundle.json` + every `site/**` file (manifest comparison catches added/removed files, not just changed ones).
- `test/e2e/golden.test.ts` + `test/golden/mini-express-app.bundle.json` — committed golden; `pnpm golden:update` script regenerates it (script prints a diff summary for PR review).
- `test/e2e/selfhost.test.ts` — run the built CLI against the repo root: exit 0, bundle schema-valid, ≥ 3 tours available, `run-report.json` has no `error` field, and every `warnings[].code` is in a short frozen allowlist (≤ 5 codes, each justified by a comment).
- `packages/viewer/test/e2e/smoke.spec.ts` (Playwright, chromium): (a) static: serve the emitted fixture site over a throwaway static server **and** load via `file://` — navigate all tours/steps, toggle theme+locale, assert zero console errors/CSP violations, and verify reduced-motion emulation disables transitions (32's behavioral check); (b) serve mode: launch `onboard serve` with `NODE_ENV=test ONBOARD_TEST_MOCK_LLM=1` (the seam owned by issue 24), run one chat round-trip, assert the token meter updates.
- `pnpm test:e2e` umbrella script (runs determinism, golden check, selfhost, and the Playwright smoke; `pnpm golden:update` regenerates).
- CI wiring for these jobs (consumed by 38).

## Detailed Requirements

1. Determinism test must run the CLI as a subprocess (not in-process) with `TZ` and
   `LANG` varied between the two runs (`TZ=UTC` vs `TZ=Asia/Tokyo`, `LANG=C` vs
   `en_US.UTF-8`) — byte equality must survive environment variation (§5.6).
2. Golden update flow: `pnpm golden:update` regenerates from the fixture and rewrites
   the committed file; the test fails with a readable unified-diff excerpt (first 40
   lines) when drift is unintentional. Golden file is the **bundle only** (site bytes
   covered by the double-run test; golden keeps review diffs human-scale).
3. Self-host runs after `pnpm build` against the working tree. The `generatedAt`
   omission path (§5.6-4) is exercised **deliberately**: copy mini-express-app to a
   temp dir, dirty one file without committing, generate, and assert `generatedAt`
   absent + `dirty: true` (CI checkouts are clean, so this cannot be left to chance).
4. Serve-mode smoke uses the env-gated mock seam **owned by issue 24**
   (`ONBOARD_TEST_MOCK_LLM=1`, honored only under `NODE_ENV=test`; 24's unit tests
   prove inertness elsewhere — this issue only consumes it).
5. Playwright: chromium only, headless, retained trace on failure (CI artifact);
   `file://` load uses `page.goto("file://…")` — module/CORS regressions surface here
   (ADR-004 guard).
6. Console-error assertion: any `console.error` or `pageerror` or CSP violation report
   fails the smoke (collect via CDP events).
7. Runtime budget: full gate set ≤ 6 min on CI.

## Acceptance Criteria

- [ ] Determinism: two env-varied runs produce identical `{path, sha256, size}` manifests (test demonstrably fails when a `Date.now()` is planted in the emitter — spot-check documented in PR, then reverted).
- [ ] Golden: committed, reviewed, and `golden:update` produces zero diff immediately after.
- [ ] Self-host: green on CI with the frozen warning-code allowlist; dirty-worktree case asserts `generatedAt` omitted + `dirty: true`.
- [ ] Playwright static smoke passes on `http://` and `file://` (incl. reduced-motion emulation check); serve smoke completes a chat round-trip via the 24 seam and the meter shows non-zero tokens.
- [ ] `pnpm test:e2e` runs all of the above locally; all gates wired as CI jobs (names: `determinism`, `golden`, `selfhost`, `viewer-smoke`) — required-check flip happens in 38.

## Validation

CI run on the PR shows all four jobs green; local `pnpm test:e2e` reproduces.

## Dependencies

27 (emitter), 32 (viewer complete), 35 (chat UI for the serve smoke — brings 33/34
transitively), 24 (mock seam), 05 (fixture).

## Non-goals

Cross-browser matrix (chromium only in v1); performance benchmarking (§13 targets are
informative); mutation testing.

## Design References

DESIGN §14.3–14.4, §5.6, §1.2, ADR-004, ADR-007.
