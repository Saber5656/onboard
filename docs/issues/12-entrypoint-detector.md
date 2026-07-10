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
   - `scripts.start`/`scripts.dev` parsing: tokenize the command; the file argument is
     the first token ending in `.ts/.js/.mjs/.cjs/.tsx` after stripping runner prefixes
     (`node`, `tsx`, `ts-node`, `nodemon`, `bun`); flag-shaped tokens (`-*`) skipped.
   - listen-signal: scan ts-morph call expressions for property-access calls named
     `listen`, or identifier calls `serve`/`createServer` — in any project file; that
     file becomes a candidate.
   - framework configs: existence of `next.config.*`, `nuxt.config.*`, `astro.config.*`
     → candidate file = the config file itself, kind `web-app`, and set
     `flowEligible: false` (framework internals are not traceable in v1 — this field
     lives on EntryPoint; add to model if absent, coordinate §5.2).
   - CLI framework: file imports `commander`/`yargs` **and** contains a `.parse(` call.
3. Kind assignment precedence when multiple signals hit one file:
   bin > server > web-app > lib.
4. `symbol`: exported function invoked in a top-level statement of the candidate file
   (e.g. `main()` / `void main()` / `main().catch(...)`); else null.
5. Config override: when `config.entryPoints` non-empty, skip detection; validate files
   exist (03 already validated) and emit evidence `["configured entry point"]`,
   score 100, `flowEligible: true`.
6. Output: dedupe by file (merge evidence, sum scores per rule table — each signal type
   counts once per file); sort score desc, tie path asc; slice `flow.maxEntries`
   (default 3); dropped candidates → warning `entries-dropped` with count.
7. Zero candidates: return empty array + warning `entries-none` (entry-flow tour becomes
   unavailable via §6.1 — pipeline concern).

## Acceptance Criteria

- [ ] mini-express-app: `src/server.ts` is the top entry with evidence containing both the listen signal and the scripts.dev parse; kind `server`.
- [ ] Planted `bin` field (`{"cli": "./src/cli.ts"}`) in a test copy outranks the server file (bin 50 > server signals when isolated) — assert ordering.
- [ ] `next.config.js` fixture-let: candidate produced with kind `web-app` and `flowEligible: false`.
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
