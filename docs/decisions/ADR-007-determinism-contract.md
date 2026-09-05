# ADR-007: Byte-level determinism contract and exact-pinning of artifact-affecting dependencies

Status: Accepted (2026-07-10)

## Context

Determinism is one of the two differentiation axes (ADR-002). "Mostly deterministic" is
not a claim that can be tested or marketed; only byte-identical output is.

## Decision

- Contract (DESIGN §5.6): for a given (worktree content, effective config, onboard
  version, locale), with LLM disabled, `tour-bundle.json` and all of `site/` are
  **byte-identical** across runs and machines (macOS/Linux).
- Normative implementation rules: key-sorted stable JSON with `<` escaping; documented
  sort for every output array; ban on `Date.now()`/`new Date()`/`Math.random()`/unsorted
  iteration in generate paths; `meta.generatedAt` = HEAD committer date, omitted when
  dirty; invariant number/date formatting.
- Dependencies whose output feeds artifacts (**shiki, markdown-it, ts-morph**) are pinned
  **exact** in `package.json` (not just lockfile), so npm consumers get the same bytes as CI.
- CI enforces: double-run hash equality + committed golden bundle (DESIGN §14.4).
  A failing determinism gate blocks merge.

## Consequences

- "Run it twice, diff nothing" becomes a demoable, CI-provable product property.
- Exact pins mean dependency updates are deliberate PRs that regenerate the golden file —
  slightly more maintenance, fully intentional.
- LLM-mode output is exempt by definition; provenance is per-step (`bodySource`), and the
  narration cache gives practical stability between cache invalidations.

## Alternatives considered

- Semantic determinism (same JSON modulo key order/whitespace): untestable with a plain
  hash, invites drift. Rejected.
- Timestamping every run (`generatedAt = now`): breaks reproducibility for zero user
  value; commit dates carry the meaningful freshness signal. Rejected.
