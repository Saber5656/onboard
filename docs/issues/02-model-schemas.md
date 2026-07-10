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
- `src/util/errors.ts` — shared error taxonomy: `UsageError`, `AnalysisError`, `SecretGateError`, each carrying a stable kebab-case `code` (exit-code mapping is wired later in issue 04; issue 03 already throws these).
- `src/util/stable-json.ts` — `stableStringify(value): string`.
- `src/util/order.ts` — `byteOrderCompare(a, b)`: the repo-wide "byte order" comparator (UTF-8 byte sequence comparison, §5.6 rule 2); `stableStringify` key sorting and every producer's array sort use it.
- `src/util/hash.ts` — `sha256Hex(input: string | Uint8Array)`, `short12`, `short8`, `excerptId(path, startLine, endLine, text)`, `entryId(file, symbol)`, `stepId(tourId, index, title)`, `secretFindingId(path, line, ruleId)`.
- `src/util/paths.ts` — `toRepoRelPosix(root, abs)` (POSIX separators, NFC normalize), `safeJoin(root, rel)` (resolve + prefix containment assert, rejects `..` escapes and absolute `rel`), `stripControlChars(s)` (§11.3: remove C0 except `\t\n\r`, and DEL).
- Unit tests for all of the above.

## Detailed Requirements

1. Schemas must match DESIGN §5.2–§5.5 field-for-field, including literal unions
   (`ModuleRole`, `TourKind`, `bodySource`) and optionality. `meta.generatedAt` is
   **omitted**, never null, when dirty: model as `.optional()` **plus** a schema
   refinement rejecting `generatedAt` present while `dirty === true` (and requiring
   ISO-8601 UTC format when present). All object schemas are `.strict()` unless the
   field is an intentional record/map (`excerpts`, `postings`).
2. Types referenced by DESIGN §5.2 but specified in detail by later issues get
   **versioned placeholder schemas here with a named owner**: `ManifestInfo`/`CiWorkflow`
   (owner: issue 07), serialized `ts` info `{ sourceFileCount, tsconfigPath }` (issue 09),
   `PipelineWarning` (fully specified in §6.1 — implement fully here), `Bm25Index`
   (owner: issue 26). Placeholders are narrow (`z.object` with the fields already named
   in DESIGN, `.strict()`), so owners extend rather than replace.
3. Path-typed string fields (`FileNode.path`, anchors, `CodeExcerpt.file`,
   `ViewerIndex.files[].p`, module paths, symbol files) share one `RepoPathSchema`:
   non-empty, no leading `/`, no `\`, no `..` segment, no control chars, NFC-normalized.
4. `SCHEMA_VERSION = 1` exported constant; `TourBundleSchema` pins `schemaVersion: z.literal(1)`.
5. `stableStringify` (§5.6 rule 1): recursively sorts object keys with
   `byteOrderCompare` (UTF-8 byte order), 2-space indent, `\n` newlines, single
   trailing newline, every
   `<` in string values emitted as `\u003c`. Arrays keep caller order (sorting arrays is
   the producer's contract, §5.6 rule 2). Throws on any `undefined` value (object
   property or array element — producers must omit instead), functions, BigInt,
   NaN/Infinity, and circular references (deterministic error, code `stable-json-invalid`).
6. Line-number invariants: `startLine`/`endLine` are positive integers with
   `endLine >= startLine` (refinement on `Anchor`, `SymbolRef`, `CodeExcerpt`).
7. Id helpers must reproduce the exact formats of §5.1 (prefixes `x`, `e`,
   `s`; `stepId` = `<tourId>/<NN>-<slug>` with ascii kebab slug, max 40 chars, `NN`
   zero-padded 2-digit).
8. `stepId` slugging: lowercase, non-alphanumerics → `-`, collapse repeats, trim `-`;
   empty slug → `step`.
9. Committed example objects: `test/model/examples.ts` exports one minimal valid literal
   per schema (RepoModel, TourBundle, each StepWidget variant, TokenReport, CodeExcerpt)
   — these serve as documentation and as the parse-fixture inputs below.
10. No dependencies beyond `zod` and `node:crypto`/`node:path`.

## Acceptance Criteria

- [ ] Every `examples.ts` literal parses through its schema; rejection tests cover: unknown keys at bundle top level, `meta`, tour, step, widget, and index-entry level; wrong literal values; zero/negative line numbers; `endLine < startLine`; `generatedAt` present while `dirty: true`; absolute / `..` / backslash / control-char path values via `RepoPathSchema`.
- [ ] Fixed test vectors pass, e.g. `short12(sha256("abc"))` equals the known sha256 prefix `ba7816bf8f01`; `excerptId`/`entryId`/`stepId` outputs are asserted against hard-coded expected strings (compute once, freeze in test).
- [ ] Property tests: `stableStringify` output is invariant under object key insertion order (≥ 100 random cases); output contains no raw `<` character; snapshot test proves `"</script>"` serializes with `\u003c` (no `</script>` substring in output); `JSON.parse(stableStringify(x))` deep-equals `x` for JSON-safe `x`.
- [ ] `stripControlChars`: removes C0 controls and DEL, preserves `\t`, `\n`, `\r` (table-driven test incl. a NUL byte).
- [ ] `byteOrderCompare`: ASCII cases match default JS ordering; a non-ASCII case where UTF-8 and UTF-16 orders differ is frozen (e.g. `"｡"` vs `"က0"`-range pair) proving UTF-8 semantics.
- [ ] `safeJoin(root, "a/../../etc/passwd")`, `safeJoin(root, "/abs")`, and a symlink-free `..` chain all throw; `safeJoin(root, "src/index.ts")` resolves inside root.
- [ ] Error classes: each carries its `code`; `instanceof` distinguishes the three types (used by 03/04).
- [ ] `pnpm --filter onboard-cli-placeholder test` green (the enumerated cases above replace any coverage-percentage gate).

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
