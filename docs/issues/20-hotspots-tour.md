# Title

Hotspots tour builder

## Summary

Implement `src/tours/hotspots.ts` per DESIGN §7.4: rank files by recency-weighted commit
frequency (half-life decay), build the tour — honest method intro, one step per top file
(excerpt + activity facts + co-change notes), closing step. Requires git stats; the
pipeline already gates availability.

## Context

Hotspots answer "where does this team actually work". The scoring must match the design
formula exactly so the fixture's scripted history produces a predictable, testable
ranking.

## Scope

- `src/tours/hotspots.ts` — builder + `hotspotScore(commits: {dateIso}[], halfLifeDays, nowRef)` pure function.
- Unit tests.

## Detailed Requirements

1. Score per file (§7.4): `Σ exp(−ln2 · ageDays / halfLifeDays)` over that file's
   commit dates (`GitStats.perFile[].commitDatesIso`, provided by issue 08; capped at 50
   dates per file — the cap slightly underestimates scores of extreme-churn files, which
   is acceptable and documented in the method step). **`nowRef` = headCommitterDateIso**
   (not wall clock — determinism §5.6); ageDays = (nowRef − commitDate) / 86400000,
   floored at 0.
2. Eligibility filter (§7.4): exclude `isTest`, `lang: null`, lockfile basenames
   (`pnpm-lock.yaml yarn.lock package-lock.json bun.lockb Cargo.lock poetry.lock`), and
   files no longer present. Rank score desc, tie path asc; take `hotspots.maxFiles`
   (default 10); fewer than 2 eligible → TourUnavailable `too-few-steps` path via
   framework.
3. Intro step: method explanation facts (halfLifeDays, commitsAnalyzed, honesty note key
   `hot.method`).
4. File steps: excerpt = declaration of the file's first indexed exported symbol, else
   file head; facts = commits count, lastTouched (ISO date only, no time), score rounded
   to 2 decimals, co-change partners for this file from GitStats.cochange (top 3 by
   count, formatted `path (N shared commits)`).
5. Closing step: pointer facts to entry-flow/architecture tours.
6. All ordering deterministic; scores computed with plain `Math.exp` on numbers derived
   from ISO strings (no Date locale parsing — use `Date.parse` on ISO-8601 only).

## Acceptance Criteria

- [ ] `hotspotScore` unit vectors: single commit at nowRef → 1.0; one commit exactly halfLife old → 0.5 (±1e-9); mixed set hand-computed.
- [ ] mini-express-app: `src/services/userService.ts` ranks #1 (fixture history is scripted for this — assert); its step lists the `db/repo.ts` co-change note with the scripted count.
- [ ] Lockfile and test files absent from steps despite having commits.
- [ ] Tour = intro + N + closing; N = min(eligible, 10).
- [ ] Double-run determinism (scores identical — nowRef is commit-anchored, not wall-clock; explicitly assert two runs 1 s apart produce identical bundles).

## Validation

`pnpm --filter onboard-cli-placeholder test` (hotspots suite).

## Dependencies

17 (framework), 08 (git stats — including the coordinated `commitDatesIso` addition).

## Non-goals

Bus-factor/ownership display (privacy §11.9 — counts only, already in facts);
complexity metrics; wall-clock-relative "recent" phrasing (everything anchored to HEAD).

## Design References

DESIGN §7.4, §6.4, §5.2 GitStats, §5.6 (nowRef anchoring), §11.9.
