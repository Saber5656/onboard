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
gives the secret gate (23) a single corpus to scan.

## Scope

- `src/tours/framework.ts` — `TourBuilder` type, `assembleTour({ kind, id, title, summary, steps }) → Tour` (validates ≥ 2 steps → else returns `TourUnavailable`), `makeStepId`, `estimatedMinutes = ceil(steps × 1.5)`.
- `src/tours/excerpts.ts` — `createExcerptRegistry(root, files)` returning `{ takeExcerpt(file, startLine, endLine, opts?) → { excerptId } | null, all() → Record<string, CodeExcerpt> }`.
- `src/tours/run-builders.ts` — executes selected builders in the fixed §5.6 order, honoring availability (16).
- Unit tests.

## Detailed Requirements

1. `takeExcerpt` rules (§7.1/§7.6): clamp span to `excerpt.maxLines` (default 40) from
   `startLine`; `opts.context: true` expands ±`contextLines` (call sites); `opts.fileHead: true`
   starts after a leading comment block when that block matches license heuristics
   (first non-empty lines are a comment containing any of `license|copyright|spdx`,
   case-insensitive) and takes `maxLines / 2` lines.
2. Reads via `safeJoin` from the FileNode set only; `lang: null` files (binary/minified)
   → return null (caller must handle); CRLF normalized to LF in `text` (determinism —
   documented; the anchor still refers to original line numbers, which are unchanged by
   EOL normalization).
3. Dedup: same (file, startLine, endLine, content) → same excerptId, registered once.
   Registry `all()` returns key-sorted record.
4. Text is decoded UTF-8 (lossy replacement for invalid bytes) then
   `stripControlChars` (preserving `\t\n`).
5. `run-builders.ts`: input = model + availability + config.tours; output =
   `{ tours: Tour[], excerpts, warnings }`; builders receive a `StringTable` (issue 22;
   until then a pass-through stub table is provided here with TODO markers) — the
   framework defines the `StringTable` TypeScript interface so 18–21 can code against it
   in parallel with 22.
6. Every step passes zod `TourStepSchema` at assembly time, in all environments —
   validation cost is negligible at ≤ ~100 steps, and failing fast beats emitting a
   malformed bundle.

## Acceptance Criteria

- [ ] `assembleTour` with 1 step returns TourUnavailable with reason `too-few-steps`; with 2+ steps produces sequential `stepId`s (`architecture/01-…`, zero-padded) and correct estimatedMinutes.
- [ ] Excerpt windowing: 100-line file from line 10 → 40 lines (10–49); context option expands a 1-line call site to 7 lines (±3); fileHead on a fixture file with an MIT header starts after the header and takes 20 lines.
- [ ] Dedup: two identical requests yield one registry entry; differing end lines yield two.
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
