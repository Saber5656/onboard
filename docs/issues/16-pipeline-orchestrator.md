# Title

Analysis pipeline orchestrator: stage contract, warnings, degradation matrix

## Summary

Implement `src/analyze/pipeline.ts` per DESIGN §6.1: run stages 06–15 in dependency
order under the uniform stage contract (pure stages, warning accumulation,
`StageSkip`/`StageFatal` semantics), assemble the full `RepoModel`, compute the
tour-availability entries from the degradation matrix, record stage timings for the
run-report, and replace the CLI `runGenerate` stub up to the model-building step.

## Context

This issue turns ten independent analyzers into one deterministic pipeline and is the
single place where "what degrades when" is decided (§6.1 matrix). Tour building and
emission (17–27) consume its output.

## Scope

- `src/analyze/pipeline.ts` — `runPipeline(root, config, logger) → { model: RepoModel, availability: TourAvailability[], timingsMs: Record<string, number>, warnings }`.
- Wire into `src/cli/generate.ts`: pipeline runs, then (until issue 27 lands) prints the §12 summary and writes `run-report.json`; the not-implemented error moves to the emit step.
- Integration tests on all four fixtures.

## Detailed Requirements

1. Stage order and data flow exactly as DESIGN §2.1/§2.2: fsscan → (manifest ∥ git ∥
   ts-loader) → (symbols ∥ imports) → entries → modules → flow-resolve/select. `--no-git`
   skips the git stage entirely (availability reason "requires git history",
   distinguishable code `no-git-flag` vs `git-unavailable`). The pipeline (which has
   both manifest and ts outputs) emits warning `ts-workspace-shallow` when
   `manifest.workspaces` is non-empty and TS analysis loaded from a root tsconfig
   (moved here from issue 09 — the loader has no manifest access).
2. Stage contract enforcement: each stage runs inside a wrapper that (a) times it,
   (b) catches non-`StageFatal` throws and converts them to warning
   `pipeline-stage-error` + documented degraded output (the §6.1 matrix row), (c) merges
   stage warnings into the model. `StageFatal` (e.g. fsscan root unreadable) aborts →
   `AnalysisError` (exit 3).
3. Degradation matrix implemented as one pure function
   `computeAvailability(model, config) → TourAvailability[]` with the §6.1 table:
   hotspots unavailable without git; entry-flow unavailable without ts or without any
   flow-eligible CallPath; contributing always available; architecture always available;
   < 5 analyzable files → warning `pipeline-tiny-repo` and only architecture guaranteed.
   `--tours` subset filters availability output (unselected → not emitted at all).
4. `RepoModel.warnings` = ordered stage warnings (stage, code, message, path?).
   Timings recorded per stage (integer ms) — timings go **only** into run-report (§12),
   never into the bundle (determinism).
5. Memory guard: after ts stages, call `project.forgetNodesCreatedInBlock`-style cleanup
   where ts-morph allows; document that the Project is released before emit stages.
6. `--json` run: stdout report includes timings, counts (files, symbols, edges,
   entryPoints, callPaths), warnings, availability.

## Acceptance Criteria

- [ ] mini-express-app: all four tours available; counts non-zero; `runGenerate` exits 3 with `not-implemented` **after** printing pipeline summary (until 27) — assert stderr order.
- [ ] plain-docs: availability = architecture + contributing available; entry-flow reason "requires TS/JS analysis in v1"; hotspots available (fixture has git history via helper).
- [ ] `--no-git` on mini-express-app: hotspots unavailable with reason code `no-git-flag`; git stage timing absent.
- [ ] Fault injection: a stage stubbed to throw (test seam) degrades per matrix and the run still completes with `pipeline-stage-error` warning; a `StageFatal` from fsscan aborts with exit 3.
- [ ] Double-run: identical serialized RepoModel (excluding in-memory ts field) and identical availability on every fixture.

## Validation

`pnpm --filter onboard-cli-placeholder test` (pipeline suite) + running the built CLI on
each materialized fixture manually once, eyeballing the summary output.

## Dependencies

06, 07, 08, 09, 10, 11, 12, 13, 14, 15 (and CLI shell 04).

## Non-goals

Tour building (17+), emission (27), parallel stage execution (v1 runs stages serially —
determinism first; revisit only with evidence).

## Design References

DESIGN §6.1, §2.1–2.2, §12, §13, §5.2 RepoModel.
