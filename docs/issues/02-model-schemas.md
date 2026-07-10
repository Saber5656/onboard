# Title

Core data model: zod schemas, stable JSON serializer, ids/hashing/path utilities

## Summary

Implement the canonical data model of DESIGN §5 in `packages/onboard/src/model/` (zod v4
schemas + inferred TS types for RepoModel, TourBundle, Tour, TourStep, CodeExcerpt,
ViewerIndex, StepWidget, TokenReport) and the determinism utilities of §5.1/§5.6 in
`src/util/` (stable JSON, sha256 ids, POSIX path handling, `safeJoin`).

## Context

Every analyzer, tour builder, and emitter produces or consumes these types. The
determinism contract (ADR-007) lives or dies in `stable-json.ts`. This issue is
pure library code with exhaustive tests and zero I/O.

## Scope

- `src/model/repo-model.ts`, `src/model/tour-bundle.ts`, `src/model/widgets.ts`, `src/model/index.ts` — zod schemas named `RepoModelSchema` etc., types via `z.infer`.
- `src/util/stable-json.ts` — `stableStringify(value): string`.
- `src/util/hash.ts` — `sha256Hex(input: string | Uint8Array)`, `short12`, `short8`, `excerptId(path, startLine, endLine, text)`, `entryId(file, symbol)`, `stepId(tourId, index, title)`, `secretFindingId(path, line, ruleId)`.
- `src/util/paths.ts` — `toRepoRelPosix(root, abs)` (POSIX separators, NFC normalize), `safeJoin(root, rel)` (resolve + prefix containment assert, rejects `..` escapes and absolute `rel`), `stripControlChars(s)` (§11.3: remove C0 except `\t\n\r`, and DEL).
- Unit tests for all of the above.

## Detailed Requirements

1. Schemas must match DESIGN §5.2–§5.5 field-for-field, including literal unions
   (`ModuleRole`, `TourKind`, `bodySource`) and optionality (`meta.generatedAt`
   **omitted**, never null, when dirty — model as `.optional()`).
2. `SCHEMA_VERSION = 1` exported constant; `TourBundleSchema` pins `schemaVersion: z.literal(1)`.
3. `stableStringify` (§5.6 rule 1): recursively sorts object keys (byte-order comparison
   on UTF-16 code units), 2-space indent, `\n` newlines, single trailing newline, every
   `<` in string values emitted as `<`. Arrays keep caller order (sorting arrays is
   the producer's contract, §5.6 rule 2). Throws on `undefined` inside arrays, functions,
   BigInt, NaN/Infinity, and circular references (deterministic error, code
   `stable-json-invalid`).
4. Id helpers must reproduce the exact formats of §5.1 (prefixes `x`, `e`,
   `s`; `stepId` = `<tourId>/<NN>-<slug>` with ascii kebab slug, max 40 chars, `NN`
   zero-padded 2-digit).
5. `stepId` slugging: lowercase, non-alphanumerics → `-`, collapse repeats, trim `-`;
   empty slug → `step`.
6. No dependencies beyond `zod` and `node:crypto`/`node:path`.

## Acceptance Criteria

- [ ] All schemas parse their own documented example values and reject: unknown keys (bundle meta), wrong literal values, negative line numbers.
- [ ] Fixed test vectors pass, e.g. `short12(sha256("abc"))` equals the known sha256 prefix `ba7816bf8f01`; `excerptId`/`entryId`/`stepId` outputs are asserted against hard-coded expected strings (compute once, freeze in test).
- [ ] Property tests: `stableStringify` output is invariant under object key insertion order (≥ 100 random cases); output contains no raw `<` character; `JSON.parse(stableStringify(x))` deep-equals `x` for JSON-safe `x`.
- [ ] `safeJoin(root, "a/../../etc/passwd")`, `safeJoin(root, "/abs")`, and a symlink-free `..` chain all throw; `safeJoin(root, "src/index.ts")` resolves inside root.
- [ ] `pnpm --filter onboard-cli-placeholder test` green; coverage of `src/util/` ≥ 95% lines.

## Validation

`pnpm -r test && pnpm typecheck && pnpm lint`. Review checklist: diff every schema field
against DESIGN §5 tables (reviewer ticks each interface).

## Dependencies

01.

## Non-goals

Producing model instances (analyzers, issues 06–16); bundle emission (27); schema docs
generation.

## Design References

DESIGN §5 (all), §11.3 (control chars), ADR-007.
