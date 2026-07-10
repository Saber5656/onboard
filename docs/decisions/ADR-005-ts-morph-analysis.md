# ADR-005: ts-morph for TS/JS deep analysis; no tree-sitter in v1

Status: Accepted (2026-07-10)

## Context

Entry-point flow tours need call resolution (call expression → in-repo declaration),
which requires type information and module resolution — not just syntax. Candidates:
raw TypeScript Compiler API, ts-morph (wrapper), tree-sitter (syntax only).

## Decision

- Use **ts-morph ^28** for the TS project loader, symbol indexer, import graph, and
  call-path tracer (DESIGN §6.6–6.10). It bundles its own TypeScript compiler, decoupling
  analysis behavior from our build-toolchain TS version.
- The v1 language-agnostic fallback (file tree, git, manifests) needs **no parser at
  all** — therefore **no tree-sitter dependency in v1**. tree-sitter enters only with v2
  multi-language analysis (DESIGN §15.2 #3).
- Analysis is parse/type-check only: analyzed code is **never imported or executed**
  (DESIGN §11, T1).

## Consequences

- Type-checker-backed call resolution — the hardest v1 feature — gets the strongest
  available foundation, with an API that delegated implementation agents can use safely
  (raw compiler API is notoriously easy to misuse).
- ts-morph is the heaviest runtime dependency; guarded by file-count/byte caps
  (DESIGN §6.6) with child-process isolation as the contingency (KU-3).
- Non-TS languages get structural-only treatment in v1 (accepted product decision;
  degradation matrix DESIGN §6.1).

## Alternatives considered

- **Raw Compiler API**: no wrapper dependency, but far higher implementation-error risk
  for delegated agents. Rejected.
- **tree-sitter now**: multi-language syntax, but no type info → cannot do call
  resolution correctly; adds WASM/native packaging load with no v1 payoff. Deferred.
