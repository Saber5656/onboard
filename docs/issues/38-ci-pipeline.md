# Title

Complete CI pipeline with all blocking gates

## Summary

Finalize `.github/workflows/ci.yml` per DESIGN §14.6: one pipeline running install →
lint/typecheck → unit → viewer build → component tests → Playwright smoke → determinism →
golden → self-host → security suite → `pnpm audit` (fail on high), on Node 22 + 24,
with sensible caching, artifacts on failure, and documentation of the required-checks
list for branch protection.

## Context

Issues 01 (skeleton), 36, and 37 created the pieces; this issue composes them into the
standing merge gate and tunes CI ergonomics (cache, timing, artifacts). Repo settings
(branch protection) are the owner's manual action — this issue delivers the exact
checklist for it.

## Scope

- Rework `.github/workflows/ci.yml`: job graph `setup → [lint, typecheck, unit(22), unit(24)] → build → [component, security, determinism, golden, selfhost] → viewer-smoke`, with `audit` (`pnpm audit --prod`) as a parallel job. Required-check names, exactly: `lint`, `typecheck`, `unit (22)`, `unit (24)`, `build`, `component`, `security`, `determinism`, `golden`, `selfhost`, `viewer-smoke`, `audit`. Job commands: `component` = `pnpm --filter viewer test`; `security` = `pnpm --filter onboard-cli-placeholder test:security` (36); `determinism`/`golden`/`selfhost`/`viewer-smoke` = the 37 scripts. The workflow also gains a `workflow_call` trigger so the release workflow (40) can reuse the full gate set.
- Caching: pnpm store cache keyed by lockfile; Playwright browser cache; built viewer assets passed between jobs via `actions/upload-artifact` (or rebuilt — choose the simpler that keeps total wall time ≤ 12 min; document the choice).
- Failure artifacts: Playwright traces, emitted hostile/fixture sites, run-reports.
- `docs/ci.md` — required-checks list + the manual branch-protection steps for the owner (default branch is already push-protected by ruleset; this adds the required status checks).
- Workflow hardening: `permissions: contents: read` at workflow level; actions pinned to major versions; no secrets consumed anywhere in CI (tests are offline by design).

## Detailed Requirements

1. Matrix: unit tests on Node 22 + 24; the heavier gates (Playwright/determinism/etc.)
   run on Node 22 only. Cross-Node-version determinism is still covered: add the cheap
   double-run bundle-hash check to the unit job so it executes on both matrix legs
   (no scheduled workflows in v1).
2. Total wall time target ≤ 12 min; measure and record in the PR.
3. `pnpm audit --prod --audit-level high` — failures block; the escape hatch is
   `pnpm.auditConfig.ignoreCves` in the root package.json (a real pnpm mechanism),
   one CVE id per entry, each requiring an ADR note.
4. Concurrency group cancels superseded runs per branch.
5. All §14.6 gates present with the exact check names from the Scope list so branch
   protection can pin them (`docs/ci.md` mirrors the list).
6. No workflow step uses `pull_request_target`, no `GITHUB_TOKEN` write scopes.
   Action pinning: `actions/*` may use major tags (documented v1 tradeoff);
   any non-first-party action (`pnpm/action-setup`) is pinned to a full commit SHA
   (§11.8).

## Acceptance Criteria

- [ ] CI green end-to-end on this issue's PR with every §14.6 gate as a distinct check.
- [ ] Failure artifact spot-check: intentionally break the viewer smoke in a scratch commit → trace uploaded; revert (evidence in PR).
- [ ] `docs/ci.md` lists the exact required-check names + owner steps; workflow-level `permissions` is read-only; `gh api repos/:owner/:repo/actions/permissions` unchanged (no auto-setting — manual owner action documented instead).
- [ ] Wall time ≤ 12 min on the PR run (link in PR description).

## Validation

The PR's own CI run is the validation. Reviewer confirms check names match `docs/ci.md`.

## Dependencies

01 (skeleton), 36, 37 (gates exist).

## Non-goals

Release workflow (40); scheduled jobs; Windows/macOS CI legs (KU-4 — post-v1);
coverage-percentage gating (coverage is reported, not gated, in v1).

## Design References

DESIGN §14.6, §11.8, ISSUE_PLAN §6.
