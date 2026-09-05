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

1. Stage order — **strictly serial, fixed** (determinism first; the §2.1 diagram shows
   data dependencies, not concurrency): `fsscan → manifest → git → ts → symbols →
   imports → entries → modules → callpaths` (call *resolution* is internal to the
   callpaths stage via issue 15; there is no separate resolve stage). `--no-git` skips
   the git stage (availability reasonCode `no-git-flag` vs `git-unavailable`). The
   pipeline emits warning `ts-workspace-shallow` when `manifest.workspaces` is
   non-empty and TS analysis loaded from a root tsconfig (it alone has both inputs).
2. Stage contract enforcement: each stage runs inside a wrapper that (a) times it,
   (b) catches throws, (c) merges stage warnings. Fatality rule: a throw from
   **fsscan** (e.g. `AnalysisError("fsscan-root-unreadable")`) aborts the run (exit 3).
   A throw from any other stage is converted to warning `pipeline-stage-error` plus the
   degraded output from this **fallback table** (normative):
   | stage | fallback |
   |---|---|
   | manifest | `manifest: null`, `readme: null`, `ciWorkflows: []` |
   | git | `git: null` |
   | ts | `ts: null` |
   | symbols / imports / entries / callpaths | empty arrays / empty edge sets |
   | modules | synthetic single module `"."` (role `unknown`, all files assigned) so the architecture tour can always build |
3. Warning sanitization at the wrapper: every warning message passes
   `stripControlChars`, absolute paths under root are relativized, and the logger's
   env-key redaction snapshot is applied — no stacks, no absolute paths, no env values
   in `PipelineWarning` (§11.2 T5/T8, §11.3).
4. Availability: `computeAvailability(model, config) → { availability: TourAvailability[],
   warnings: PipelineWarning[] }` — pure — implementing the §6.1 matrix with the §5.3
   shape `{ kind, available, reason, reasonCode }`: hotspots without git
   (`no-git-flag`/`git-unavailable`); entry-flow without ts (`no-ts`) or with zero
   surviving CallPaths (`no-traceable-entries`); architecture/contributing always
   available at this stage (builders may still return unavailable, merged later by 17);
   < 5 analyzable files → warning `pipeline-tiny-repo` (architecture is the only
   *guaranteed* tour; others keep their own rules). `--tours` subset filters
   availability output (unselected → not emitted at all).
5. `RepoModel.warnings` = ordered stage warnings (stage, code, message, path?).
   Timings recorded per stage (integer ms), keys exactly the stage names of rule 1 —
   timings go **only** into run-report (§12), never into the bundle (determinism).
6. ts-morph lifecycle: the `Project` lives only from the ts stage through the callpaths
   stage; after callpaths, the pipeline drops every reference (serialized
   `RepoModel.ts` carries only `{ sourceFileCount, tsconfigPath }`); a test asserts the
   returned model stableStringify-serializes without error (proves no live compiler
   objects escape).
7. `--json` run: stdout report matches the §12 run-report shape exactly (counts incl.
   entryPoints/callPaths/steps-so-far, availability, warnings, stageTimingsMs).

## Acceptance Criteria

- [ ] mini-express-app: all four tours available; counts non-zero; `runGenerate` exits 3 with `not-implemented` **after** printing pipeline summary (until 27) — assert stderr order.
- [ ] plain-docs (4 files, git history from helper): warning `pipeline-tiny-repo` present; availability = architecture + contributing + hotspots available, entry-flow unavailable with `reasonCode: "no-ts"` (asserts the §6.1 "guaranteed vs own-rules" semantics).
- [ ] `--no-git` on mini-express-app: hotspots unavailable with `reasonCode: "no-git-flag"`; `stageTimingsMs` has no `git` key; the key set equals the executed stage names exactly.
- [ ] Fault injection (test seam stubs): manifest stage throw → run completes, `manifest: null`, `pipeline-stage-error` warning; modules stage throw → synthetic `"."` module; fsscan throw → exit 3.
- [ ] Warning sanitization: a stage throw whose message contains an absolute path and a planted env key value surfaces relativized and redacted in the run-report.
- [ ] `--json`: stdout parses as the §12 shape (single JSON document; availability + counts fields present); nothing else on stdout.
- [ ] Double-run: identical serialized RepoModel (ts field = derived data only, proven serializable) and identical availability on every fixture.

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
