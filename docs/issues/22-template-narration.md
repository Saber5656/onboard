# Title

Template narration engine with en/ja string tables

## Summary

Implement `src/narrate/templates/` per DESIGN §8.1 and §7.6: a typed `StringTable`
(key → typed-facts → markdown-subset string) with complete English and Japanese tables,
invariant formatting helpers, and the tone/fact-discipline rules — replacing the
framework's stub table so every tour step body renders real prose in both locales.

## Context

Template narration is the deterministic default voice of the product (ADR-002) and the
draft input for LLM enhancement (25). Type-checked exhaustive key parity keeps en/ja
from drifting.

## Scope

- `src/narrate/templates/keys.ts` — the `StringTable` interface: one method per step
  type with its typed facts parameter (final key set: `arch.welcome/map/module/deps/next`,
  `flow.overview/hop/recap/truncated`, `hot.method/file/closing`,
  `contrib.welcome/setup/test/lint/ci/conventions/start/generic`, plus shared
  `common.unavailableTour`, `common.branchNotes` — adjust only with builder issues'
  agreement, keys are referenced by 18–21).
- `src/narrate/templates/en.ts`, `ja.ts` — implementations.
- `src/narrate/format.ts` — `formatCount`, `formatIsoDate` (date part only), `formatList`
  (Oxford comma en / 「、」 ja), all locale-invariant per §8.1.
- Unit tests + snapshot tests for both locales.

## Detailed Requirements

1. Exhaustiveness: `en` and `ja` both implement the full `StringTable` interface —
   a missing key is a TypeScript compile error (no runtime fallback between locales).
2. Output constraints per key (validated by tests): markdown subset only (no raw HTML —
   test asserts no `<` followed by ASCII letter), ≤ 140 words (en) / ≤ 300 chars (ja)
   per body, identifiers wrapped in backticks, internal links only of the form
   `[title](#/tour/<id>/step/<n>)`.
3. Fact discipline: template functions may interpolate **only** fields of their typed
   facts argument (code-review rule + a lint-ish test: templates run against
   fully-populated fake facts and against minimal facts — no `undefined` leaking into
   output; every optional fact has an explicit conditional).
4. Tone (documented in `keys.ts` header): friendly-precise; no marketing language; honest
   hedges verbatim where designed (hotspots method note: "frequency ≠ importance, but
   correlates"; ja: 「変更頻度は重要度そのものではありませんが、よい相関があります」).
5. ja specifics: full-width punctuation, no literal translations of code terms (keep
   `entry point` → 「エントリポイント」, module → 「モジュール」), numbers via the same
   invariant helpers (ASCII digits).
6. The framework (17) swaps its stub for this table; builders 18–21 need zero changes
   (interface already fixed by 17) — verify by running their suites unchanged.

## Acceptance Criteria

- [ ] Both locales implement every key (compile-time); snapshot tests exist for every key in both locales with representative facts.
- [ ] mini-express-app full generation (pipeline + builders + this table): every step body is non-empty, HTML-free, and within length limits, in both `--locale en` and `--locale ja` runs.
- [ ] `formatIsoDate("2026-01-05T09:00:00Z")` → `2026-01-05` regardless of process TZ (test sets `TZ=Asia/Tokyo` and `TZ=UTC`).
- [ ] Word/char limits enforced by a table-driven test over all snapshots.
- [ ] Builders' existing tests pass unchanged after the stub swap.

## Validation

`pnpm --filter onboard-cli-placeholder test` (narration suite + all tour suites);
human review of ja snapshots by the repo owner is tracked in issue 41 (KU-7), not here.

## Dependencies

17 (interface + stub), 18–21 (key consumers — land before or together; key set is fixed
by this issue's `keys.ts` and referenced by name in those builders).

## Non-goals

LLM rewriting (25); viewer UI strings (32 — separate table in the viewer package);
locales beyond en/ja.

## Design References

DESIGN §8.1, §7.6, §5.3 (body constraints), ADR-002.
