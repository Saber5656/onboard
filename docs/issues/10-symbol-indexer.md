# Title

Exported symbol indexer

## Summary

Implement `src/analyze/symbols/` per DESIGN §6.7: walk every source file of the loaded
ts-morph project and produce sorted `SymbolRef[]` for **exported** declarations
(functions, classes, function-typed consts, enums, interfaces, type aliases) with
one-line signatures and JSDoc summaries, capped at 40 symbols per file.

## Context

Symbols feed entry-point evidence (12), call-path endpoints (14), module `topSymbols`
(13), architecture-tour excerpts (18), the chat retrieval corpus (26), and the viewer
symbol list. Signature strings appear verbatim in narration — they must be short, clean,
and control-char-free.

## Scope

- `src/analyze/symbols/index.ts` — `indexSymbols(root, ts, files, logger) → { symbols: SymbolRef[], warnings }` (root + FileNode set needed to emit repo-relative POSIX paths and deterministic order).
- `src/analyze/symbols/signature.ts` — signature formatting helpers.
- Unit tests on fixtures + inline snippets.

## Detailed Requirements

1. Exported detection per file: `export function/class/enum/interface/type`,
   `export const X = <arrow|function expression>`, `export default <fn|class>`,
   `export { A, B }` re-export lists resolve to the local declaration when it is in the
   same file; cross-file re-exports are skipped with warning `symbols-reexport-skipped`
   (avoids double counting; the defining file already reports the symbol).
2. Default exports: `SymbolRef.name` = file basename without extension; signature is
   prefixed `default` — anonymous default function → `default function <basename>(…)`,
   named default `export default function main()` keeps `name: "main"` and signature
   `default function main()`.
3. CJS detection (degradation rule): a file containing top-level `module.exports =` or
   `exports.<name> =` assignments gets one warning `symbols-cjs-unsupported`; CJS
   exports are not indexed in v1. A mixed file (ESM `export` + CJS assignments) still
   indexes its ESM exports and also warns.
4. `SymbolRef` fields (§5.2): `file` (repo-relative POSIX via `toRepoRelPosix`), `name`,
   `kind` (`function|class|const|enum|interface|type`), `startLine`/`endLine` (1-based,
   declaration span **excluding leading JSDoc**), `signature`, `jsdocSummary`,
   `exported: true`. Non-exported declarations are **not** indexed in v1.
5. Signature formatting: declaration header text up to (not including) the body —
   e.g. `function createUser(input: NewUser): Promise<User>` — whitespace collapsed to
   single spaces, ≤ 120 chars then `…` appended. Per kind: classes
   `class UserService extends Base` (heritage included); consts
   `const parseId = (raw: string) => number` (type text when arrow body elided);
   interfaces `interface Foo extends Bar`; type aliases `type Result = <RHS text>`
   (RHS included, same 120-char truncation); enums `enum Status (4 members)`.
6. Sanitization: every emitted string (`name`, `signature`, `jsdocSummary`, `file`)
   passes `stripControlChars` (§11.3) — signatures/JSDoc reach narration verbatim.
7. `jsdocSummary`: first JSDoc block's description, first sentence only (split on `.`
   followed by space/EOL), ≤ 200 chars, tags ignored; absent → null.
8. Ordering: sort by (repo-relative file path byte order, startLine asc), applied
   **after** the per-file cap; the cap itself keeps the first 40 by startLine within
   each file (drop the rest, warning `symbols-file-capped` with file + dropped count).
9. Robustness: any per-node ts-morph throw is caught → warning `symbols-node-error`
   (file + line), continue. The indexer never throws for content reasons.

## Acceptance Criteria

- [ ] mini-express-app: `src/services/userService.ts` exports are all indexed with correct kinds/lines; handler function in `src/routes/users.ts` present; signatures match hard-coded expected strings for ≥ 3 symbols (frozen in test), including one interface and one type alias.
- [ ] Anonymous `export default function` snippet yields name = file basename and signature `default function <basename>(…)`; a documented declaration's `startLine` is the declaration line, not the JSDoc start.
- [ ] JSDoc summary extracted for a documented fixture symbol; multi-sentence JSDoc keeps only the first sentence; hostile JSDoc containing C0 control chars and `<script>` text is emitted control-char-free (the `<` itself survives — HTML safety is the emitter's context, §11.5).
- [ ] File with 45 exported consts (generated in test) → the first 40 by startLine indexed + `symbols-file-capped`.
- [ ] js-lib (`module.exports = { parse }`): warning `symbols-cjs-unsupported` once per file, zero symbols for it; a mixed ESM+CJS snippet indexes the ESM export and still warns.

## Validation

`pnpm --filter onboard-cli-placeholder test` (symbols suite). Determinism: two runs on
mini-express-app produce identical serialized arrays.

## Dependencies

09.

## Non-goals

Non-exported symbols; CJS `module.exports` parsing (documented degradation; revisit v2);
call graphs (14); rename/reference tracking.

## Design References

DESIGN §6.7, §5.2 SymbolRef, §6.9/§7.2/§10.4 (consumers).
