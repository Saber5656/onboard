# Title

Contributing tour builder

## Summary

Implement `src/tours/contributing.ts` per DESIGN §7.5: the always-available (degrading)
tour that turns manifest/CI/docs facts into a first-contribution walkthrough — welcome &
prerequisites, setup, run & test (with runner-specific single-test commands), lint
tooling, CI overview, conventions, and a "where to start" step from git activity.

## Context

This tour is the OSS-adoption hook (scenario S2) and must work on repos with no code
analysis at all — it consumes only manifest (07), git (08), docs files (06), and modules
(13).

## Scope

- `src/tours/contributing.ts` — builder.
- `src/tours/runner-commands.ts` — test-runner → command-template table.
- Unit tests on all fixtures.

## Detailed Requirements

1. Step sequence §7.5 (1–7); each step included only when its facts exist; framework's
   ≥ 2-step rule guarantees welcome + at least one more (welcome itself needs no data —
   plus repo-docs/README step derived from docsFiles as the guaranteed second step;
   make that explicit: step "docs & conventions" renders whenever README exists, which
   fs-scan guarantees to detect when present; a repo with zero recognizable facts still
   yields welcome + generic "explore the tree" step — StringTable key `contrib.generic`).
2. Prerequisites facts: packageManager (with install command from table:
   pnpm→`pnpm install`, yarn→`yarn`, bun→`bun install`, npm→`npm install`; null →
   ecosystem note from otherEcosystems or generic), nodeVersion raw string.
3. Run & test: scripts table (dev/build/test/lint as present); single-test command from
   runner table: vitest → `pnpm vitest run <path>` (pm-adjusted prefix: npm→`npx`,
   yarn→`yarn`, bun→`bunx`), jest → `<px> jest <path>`, mocha → `<px> mocha <path>`,
   go → `go test ./<pkg>/...`, pytest → `pytest <path>::<test>` (go/pytest only when
   otherEcosystems indicates); test directory hint = modules with role tests (paths).
4. Lint step: linters detected list + config file presence (`eslint.config.*`,
   `.prettierrc*`, `biome.json`) from FileNode set.
5. CI step: workflow names + trigger keys + job names (07 output verbatim).
6. Conventions step: CONTRIBUTING.md first paragraph (via excerpt service `fileHead`
   on the markdown file, maxLines 15) when present; else README-based fallback; CoC
   presence noted in facts.
7. Where to start (§7.5 item 7): top-3 modules by count of `commitDatesIso` entries
   within 90 days before headCommitterDateIso, summed across module files
   (`GitStats.perFile[].commitDatesIso` from issue 08), excluding roles
   {infra, build, tests, docs}; tie-break module path asc; each with rationale facts
   (recent-commit count, role). No git → step omitted.
8. Facts must never invent commands: only table-derived or manifest-verbatim strings.

## Acceptance Criteria

- [ ] mini-express-app: steps 1–7 all present; single-test command is exactly `pnpm vitest run test/users.test.ts`-shaped (template + real path from tests module); CI step lists the fixture's 2 job names; where-to-start includes `src/services` (scripted history) with rationale.
- [ ] js-lib (npm, no CI, no CONTRIBUTING): setup uses `npm install`; CI and conventions steps omitted; tour still ≥ 4 steps.
- [ ] plain-docs (no manifest): welcome + docs/conventions (+ generic) — tour available with ≥ 2 steps; no fabricated commands anywhere (grep facts for `install` — only from the table when pm known).
- [ ] Runner table unit tests: one case per (runner × package manager) combination in the table.
- [ ] Double-run determinism.

## Validation

`pnpm --filter onboard-cli-placeholder test` (contributing suite); reviewer reads the
mini-express-app step facts end-to-end for command correctness.

## Dependencies

17 (framework), 07 (manifest), 08 (git), 13 (modules).

## Non-goals

Issue-tracker integration ("good first issue" labels — v2); executing any command;
per-workspace instructions (v2 monorepo).

## Design References

DESIGN §7.5, §6.3, §6.4, §5.3.
