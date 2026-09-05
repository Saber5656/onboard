# Title

Entry-point detector with evidence scoring

## Summary

Implement `src/analyze/entries/` per DESIGN §6.9: gather entry-point candidates from
manifest fields, script commands, framework markers, symbol-level server signals, and
filename conventions; score them with the normative table; return the top
`flow.maxEntries` as `EntryPoint[]` with human-readable evidence strings. Config
`entryPoints` overrides detection entirely.

## Context

Entry points seed the call-path tracer (14–15) and the entry-flow tours (19). Evidence
strings appear verbatim in tour narration, so they must be precise and readable.

## Scope

- `src/analyze/entries/index.ts` — `detectEntryPoints(model-parts, config, logger) → { entryPoints: EntryPoint[], warnings }` (inputs: manifest (07), symbols (10), imports (11), files (06), ts project (09)).
- Unit tests on fixtures.

## Detailed Requirements

1. Scoring table exactly DESIGN §6.9 (bin +50, main/exports +30, scripts.start/dev file
   +25, listen/serve/createServer call +20, framework config +20, conventional filename
   +10, CLI framework usage +15). A candidate accumulates scores from multiple signals.
2. Signal extraction details:
   - `scripts.start`/`scripts.dev` parsing — deterministic tokenizer subset: split on
     whitespace honoring single/double quotes; drop leading `VAR=value` env
     assignments; skip runner/manager tokens (`node tsx ts-node nodemon bun pnpm npm
     yarn npx bunx exec run dev`) and flag-shaped tokens (`-*`, plus the value token
     after `--exec`); the file argument is the first remaining token ending in
     `.ts/.tsx/.js/.mjs/.cjs`. No match (shell operators `&&`, subshells, unknown
     forms) → no signal from this source.
   - listen-signal: scan ts-morph call expressions for property-access calls named
     `listen`, or identifier calls `serve`/`createServer` — only in files with
     `isTest: false` and `lang !== null` (never tests/binaries); that file becomes a
     candidate.
   - framework configs: existence of `next.config.*`, `nuxt.config.*`, `astro.config.*`
     → candidate file = the config file itself, kind `web-app`,
     `flowEligible: false` (§5.2 field; framework internals are not traceable in v1).
   - CLI framework: file has an import declaration for `commander` or `yargs` (scan
     ts-morph import declarations directly — issue 11's aggregated externals are not
     per-file) **and** contains a `.parse(` call. `isTest` files excluded.
3. All detected entries have `flowEligible: true` except framework configs (false).
   Kind assignment precedence when multiple signals hit one file:
   bin > server > web-app > lib.
4. Evidence strings are built from **fixed labels + normalized repo-relative paths
   only** — never raw script text or manifest values (§11.3; they render verbatim in
   narration). Canonical forms (exact): `package.json bin "<name>"`,
   `package.json main/exports target`, `scripts.<name> runs <path>`,
   `calls .listen()/serve()/createServer()`, `framework config <basename>`,
   `conventional filename`, `CLI framework usage (commander|yargs)`,
   `configured entry point`. All evidence strings pass `stripControlChars`.
5. `symbol` detection — exact patterns over the candidate file's top-level statements:
   an expression statement of form `f()`, `void f()`, `f().catch(…)`, or `await f()`
   where `f` is an identifier bound to a function/const declaration **exported from the
   same file**; take the first match in source order; `export default function name()`
   also qualifies (symbol = its name). Anything else → null.
6. Config override: when `config.entryPoints` non-empty, skip detection; files already
   validated by 03; evidence `["configured entry point"]`, score 100,
   `flowEligible: true`, kind from config.
7. Output: dedupe by file (merge evidence, sum scores per rule table — each signal type
   counts once per file); sort score desc, tie path asc; slice `flow.maxEntries`
   (default 3); dropped candidates → warning `entries-dropped` with count.
8. Zero candidates: return empty array + warning `entries-none` — the pipeline (16)
   maps this to the §6.1 row "no traceable entry points".

## Acceptance Criteria

- [ ] mini-express-app: `src/server.ts` is the top entry with evidence containing both the listen signal and the scripts.dev parse (exact canonical strings asserted); kind `server`; test files produce no candidates.
- [ ] Score-table unit tests: one isolated fixture per signal type asserting its exact score contribution; a combined fixture (`bin` = 50 vs server file with scripts+listen+filename = 55) asserts the resulting order with hand-computed totals — the AC asserts exact scores, not just ranking.
- [ ] Tokenizer table tests: quoted path, env-prefix (`NODE_ENV=x node src/a.ts`), `nodemon --exec tsx src/server.ts`, `pnpm tsx src/main.ts`, and an unsupported `a && b` form (no signal).
- [ ] `next.config.js` fixture-let: candidate produced with kind `web-app` and `flowEligible: false`.
- [ ] Symbol patterns: `main()`, `void main()`, `main().catch(e => …)` each detect exported `main`; non-exported or aliased callee → null.
- [ ] Config override with `{ file: "src/routes/health.ts" }` yields exactly one EntryPoint with score 100 and skips detection (no listen-derived entries).
- [ ] plain-docs: empty result + `entries-none`.

## Validation

`pnpm --filter onboard-cli-placeholder test` (entries suite); print the evidence strings
for mini-express-app and review them for narration-readability (reviewer judgment).

## Dependencies

07, 10, 11.

## Non-goals

Tracing (14–15); monorepo per-package entries (v2); framework-internal route
enumeration.

## Design References

DESIGN §6.9, §5.2 EntryPoint, §7.3 (consumer), §6.1 (degradation).
