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

- `src/analyze/symbols/index.ts` — `indexSymbols(ts: TsAnalysis, logger) → { symbols: SymbolRef[], warnings }`.
- `src/analyze/symbols/signature.ts` — signature formatting helpers.
- Unit tests on fixtures + inline snippets.

## Detailed Requirements

1. Exported detection per file: `export function/class/enum/interface/type`,
   `export const X = <arrow|function expression>`, `export default <fn|class>` (name =
   `default` → use file basename without extension as display name, flag in signature),
   `export { A, B }` re-export lists resolve to the local declaration when it is in the
   same file; cross-file re-exports are skipped with warning `symbols-reexport-skipped`
   (avoids double counting; the defining file already reports the symbol).
2. `SymbolRef` fields (§5.2): `file` (relative POSIX), `name`, `kind`
   (`function|class|const|enum|interface|type`), `startLine`/`endLine` (1-based,
   declaration span incl. JSDoc excluded), `signature`, `jsdocSummary`, `exported: true`.
   Non-exported declarations are **not** indexed in v1.
3. Signature formatting: declaration header text up to (not including) the body —
   e.g. `function createUser(input: NewUser): Promise<User>` — whitespace collapsed to
   single spaces, ≤ 120 chars then `…` appended, control chars stripped. Classes:
   `class UserService extends Base` (heritage clause included). Consts:
   `const parseId = (raw: string) => number` (use type text when arrow body elided).
4. `jsdocSummary`: first JSDoc block's description, first sentence only (split on `.`
   followed by space/EOL), ≤ 200 chars, tags ignored; absent → null.
5. Ordering: (file byte order, startLine asc). Per-file cap 40 (drop the rest, warning
   `symbols-file-capped` with file + dropped count).
6. Robustness: any per-node ts-morph throw is caught → warning `symbols-node-error`
   (file + line), continue. The indexer never throws for content reasons.

## Acceptance Criteria

- [ ] mini-express-app: `src/services/userService.ts` exports are all indexed with correct kinds/lines; handler function in `routes/users.ts` present; signatures match hard-coded expected strings for ≥ 3 symbols (frozen in test).
- [ ] `export default function` in a temp snippet yields name = file basename and a signature starting `function`.
- [ ] JSDoc summary extracted for a documented fixture symbol; multi-sentence JSDoc keeps only the first sentence.
- [ ] File with 45 exported consts (generated in test) → 40 indexed + `symbols-file-capped`.
- [ ] js-lib (CommonJS-style `module.exports = { parse }`): CJS export detection is **out of scope** — assert warning `symbols-cjs-unsupported` is emitted once per file using `module.exports`, and zero symbols for it (documented degradation).

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
