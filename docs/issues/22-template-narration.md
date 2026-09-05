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

- `src/narrate/templates/keys.ts` — **replaces the stub created by issue 17 at this
  same path** (public import path unchanged; builders 18–21 need no edits). It defines:
  - `StepBodyTable` — one method per step body: `arch.welcome/map/module/deps/next`,
    `flow.overview/hop/recap/truncated`, `hot.method/file/closing`,
    `contrib.welcome/setup/test/lint/ci/conventions/start/generic`;
  - `FragmentTable` — reusable in-body fragments: `common.branchNotes` (consumed
    inside `flow.hop`); no `common.unavailableTour` key — unavailable-tour reasons are
    plain facts of `arch.next`;
  - **named, exported fact types** for every key (builders import these to construct
    facts — the fact contract lives here, not in prose);
  - escaping helpers templates MUST use for repo-derived values: `inlineCode(v)`
    (backtick-safe), `markdownText(v)` (escapes markdown specials), and
    `internalTourLink(title, tourId, stepIndex)` producing the canonical
    `#/tour/<tourId>/step/<n>` form (§8.1/§9.2).
- `src/narrate/templates/en.ts`, `ja.ts` — implementations of both tables.
- `src/narrate/format.ts` — `formatCount`, `formatIsoDate` (date part only), `formatList`
  (Oxford comma en / 「、」 ja), `countEnglishWords(markdown)` (strip markdown
  delimiters, split on whitespace), `countJapaneseChars(markdown)` (rendered text
  chars, whitespace excluded) — all locale-invariant per §8.1.
- `src/narrate/assert-no-html.ts` — shared `assertNoRawHtmlLikeMarkup(s)` rejecting
  `<` followed by an ASCII letter **or** `/` (used by these tests and by issue 25's
  LLM-output validation — one rule, two callers).
- Unit tests + snapshot tests for both locales.

## Detailed Requirements

1. Exhaustiveness: `en` and `ja` both implement the full `StepBodyTable` +
   `FragmentTable` — a missing key is a TypeScript compile error (no runtime fallback
   between locales).
2. Output constraints per key (validated by tests): markdown subset only (no raw HTML —
   `assertNoRawHtmlLikeMarkup` over every snapshot), ≤ 140 words (en, via
   `countEnglishWords`) / ≤ 300 chars (ja, via `countJapaneseChars`) per body,
   identifiers wrapped in backticks, internal links only via `internalTourLink`
   (`#/tour/<tourId>/step/<n>`).
3. Fact discipline: template functions may interpolate **only** fields of their typed
   facts argument, and repo-derived fields **only** through `inlineCode`/`markdownText`
   (raw interpolation of untrusted facts is forbidden — a fact value like
   `[x](https://evil)` or `**bold**` must render inert; test included). Templates run
   against fully-populated and minimal fake facts — no `undefined` leaking; every
   optional fact has an explicit conditional.
4. Tone (documented in `keys.ts` header): friendly-precise; no marketing language; honest
   hedges verbatim where designed (hotspots method note: "frequency ≠ importance, but
   correlates"; ja: 「変更頻度は重要度そのものではありませんが、よい相関があります」).
5. ja specifics — enforced by an automated ja-lint test, not just review: full-width
   punctuation (`。、`), ASCII digits via the invariant helpers, and a
   forbidden-literal-translation list (e.g. `入口点` for entry point — use
   「エントリポイント」; `熱点` for hotspot — use 「ホットスポット」) asserted absent
   from all ja snapshots.
6. This issue replaces the 17 stub in place; builders 18–21 need zero changes. When
   18–21 are already merged, their suites must pass unchanged; when not, validate
   against stub-consumer tests exercising every key.

## Acceptance Criteria

- [ ] Both locales implement every key (compile-time); snapshot tests exist for every key in both locales with representative facts; `assertNoRawHtmlLikeMarkup` passes on all snapshots (and rejects a planted `</script>` and `<img` case).
- [ ] Injection test: a fact containing `[x](https://evil)` and `**bold**` renders as literal text via `markdownText`; an identifier fact containing a backtick renders safely via `inlineCode`.
- [ ] mini-express-app full generation (pipeline + builders + this table): every step body is non-empty, HTML-free, and within length limits, in both `--locale en` and `--locale ja` runs.
- [ ] `formatIsoDate("2026-01-05T09:00:00Z")` → `2026-01-05` regardless of process TZ (test sets `TZ=Asia/Tokyo` and `TZ=UTC`).
- [ ] Word/char limits enforced by a table-driven test over all snapshots using the two counting helpers; ja-lint (punctuation, digits, forbidden translations) passes.
- [ ] Builders' suites (18–21, where merged) pass unchanged after the stub swap.

## Validation

`pnpm --filter onboard-cli-placeholder test` (narration suite + all tour suites);
human review of ja snapshots by the repo owner is tracked in issue 41 (KU-7), not here.

## Dependencies

17 (interface location + stub). Runs in parallel with 18–21 (they consume the key set
via the 17 interface; their suites re-run unchanged when this lands after them).

## Non-goals

LLM rewriting (25); viewer UI strings (32 — separate table in the viewer package);
locales beyond en/ja.

## Design References

DESIGN §8.1, §7.6, §5.3 (body constraints), ADR-002.
