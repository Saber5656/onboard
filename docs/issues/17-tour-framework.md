# Title

Tour builder framework and shared excerpt service

## Summary

Implement `src/tours/` shared infrastructure per DESIGN §7.1: the `buildTour` contract
(pure builders over RepoModel), tour assembly helpers (ids, step numbering, estimated
minutes, ≥ 2-step rule), and the excerpt service (`takeExcerpt`) with windowing rules,
license-header skipping, dedup by excerptId, and the bundle-wide excerpt registry.

## Context

Four tour builders (18–21) plug into this framework; the emitter (27) consumes the
excerpt registry. Centralizing excerpt extraction keeps caps and dedup consistent and
gives the secret gate one uniform corpus of excerpt texts (the *full* gate corpus —
narration bodies, README paragraph, manifest-derived strings — is assembled by
issues 23/27).

## Scope

- `src/tours/framework.ts` — `TourBuilder` type (single-tour kinds return `Tour | TourUnavailable`; **entry-flow is the one multi-tour kind** — its builder returns `{ tours: Tour[], unavailable: TourUnavailable | null }` and run-builders handles it specially), `assembleTour({ kind, id, title, summary, steps }) → Tour | TourUnavailable` (≥ 2 steps rule; `TourUnavailable = { kind, reason, reasonCode }`), `makeStepId`, `estimatedMinutes = ceil(steps × 1.5)`.
- `src/tours/excerpts.ts` — `createExcerptRegistry(root, files)` returning `{ takeExcerpt(file, startLine, endLine, opts?) → { excerptId, file, startLine, endLine } | null` (the **final clamped/expanded span**, which callers use for `TourStep.anchor`), `all() → Record<string, CodeExcerpt> }`.
- `src/tours/run-builders.ts` — `runBuilders(model, availability, config, strings) → { tours, availability, excerpts, warnings }`: executes selected builders in the fixed §5.6 order for kinds marked available; builder `TourUnavailable` results **update** the availability entries (merged output is what the emitter serializes into `meta.tourAvailability`).
- `src/narrate/templates/keys.ts` — created **here** with the `StringTable` interface and a pass-through stub implementation (issue 22 replaces the stub tables without changing the import path — DESIGN §8.1 location).
- Unit tests.

## Detailed Requirements

1. `takeExcerpt` rules (§7.1/§7.6): clamp span to `excerpt.maxLines` (default 40) from
   `startLine`; `opts.context: true` expands ±`contextLines` (call sites);
   `opts.fileHead: true` starts after a leading license header and takes
   `maxLines / 2` lines. License-header rule (exact): skip a first-line shebang
   (`#!…`); then a "leading comment block" = consecutive lines that are `//` comments,
   or one `/* … */` block, or consecutive `#` lines (non-JS langs), ending at the first
   blank/non-comment line; skip it **only** when its text matches
   `/license|copyright|spdx/i`; otherwise start at line 1.
2. Reads via `safeJoin` from the FileNode set only: a requested path absent from the
   FileNode set, absolute, or containing `..` returns **null without any disk access**;
   `lang: null` files (binary/minified) → null; CRLF normalized to LF in `text`
   (determinism — the anchor still refers to original line numbers, which EOL
   normalization does not change).
3. Dedup happens on the **final normalized excerpt** (file, clamped startLine/endLine,
   normalized text) — identical final spans share one registry entry and one excerptId
   (§5.1 derives the id from exactly these fields). Registry `all()` returns key-sorted
   record.
4. Text is decoded UTF-8 (lossy replacement for invalid bytes) then
   `stripControlChars` (preserving `\t\n`).
5. `runBuilders` merge rule: an available kind whose builder returns `TourUnavailable`
   flips its availability entry to `{ available: false, reason, reasonCode }`; tours
   keep the §5.6 fixed order; entry-flow tours sorted by entryId.
6. Every step passes zod `TourStepSchema` at assembly time, in all environments —
   validation cost is negligible at ≤ ~100 steps, and failing fast beats emitting a
   malformed bundle.

## Acceptance Criteria

- [ ] `assembleTour` with 1 step returns TourUnavailable (`too-few-steps`); with 2+ steps produces sequential `stepId`s (`architecture/01-…`, zero-padded) and correct estimatedMinutes; an invalid step (schema violation) fails assembly with a deterministic error; a valid minimal step passes.
- [ ] Excerpt windowing: 100-line file from line 10 → lines 10–49 and the returned span says so; context option expands a 1-line call site to 7 lines; fileHead on a file with an MIT `/* */` header and on one with `//` header starts after the header; a shebang line is skipped; a non-license leading comment is kept.
- [ ] Dedup: two requests whose spans clamp to the same final range yield one registry entry; genuinely different final ranges yield two.
- [ ] Path safety: requests for `../outside`, an absolute path, and a path not in the FileNode set each return null with zero fs reads (spy on the read function).
- [ ] `runBuilders`: a builder stub returning TourUnavailable flips the merged availability entry (asserted end-to-end with a fake availability input).
- [ ] CRLF fixture file: `text` contains no `\r`; startLine/endLine match the original file's numbering.
- [ ] Binary/minified file request returns null without throwing.

## Validation

`pnpm --filter onboard-cli-placeholder test` (tours-framework suite).

## Dependencies

16 (RepoModel/availability shapes; runs after pipeline exists).

## Non-goals

Any concrete tour content (18–21); narration text (22); highlighting (27 — the registry
stores raw text only).

## Design References

DESIGN §7.1, §7.6, §5.3 (Tour/TourStep/CodeExcerpt), §5.1 (ids), §5.6.
