# Title

Secret scanner and fail-closed emit gate

## Summary

Implement `src/secretscan/` per DESIGN §11.4 and ADR-006: the rule table (7 block rules +
3 warn rules), the scanner over bundle-bound strings, finding ids/masking, the allowlist
mechanism, the exit-4 gate wiring, and the shared `rules.ts` that issue 03's config check
refactors onto.

## Context

This is the control that makes "publish your tour" a defensible default (T2). It must be
precise (metadata: rule ids, masked output) and fail closed.

## Scope

- `src/secretscan/rules.ts` — `BLOCK_RULES: { id, regex }[]`, `WARN_RULES` (regex or entropy fn), exported for reuse.
- `src/secretscan/scan.ts` — `scanStrings(items: { text, path, line? }[]) → Finding[]` and `shannonEntropy(s)`.
- `src/secretscan/gate.ts` — `enforceGate(findings, allowlist) → { blocked: Finding[], warned: Finding[] }`; blocked non-empty → `SecretGateError` (exit 4).
- Refactor: `src/config/load.ts` key-in-config check now imports `BLOCK_RULES` (removing the duplicated regex list from issue 03).
- Unit tests (table-driven positives/negatives per rule).

## Detailed Requirements

1. Rules exactly as the DESIGN §11.4 tables (ids, patterns, severities). Entropy rule:
   candidate tokens = `[A-Za-z0-9+/=_-]{32,}` sequences; compute Shannon entropy over
   chars; > 4.0 bits ⇒ finding, **except** when the source `path` has `isTest: true` or
   the surrounding 40 chars contain `sha256|integrity|hash` (case-insensitive).
2. `Finding = { id, ruleId, severity: "block"|"warn", path, line, masked }`;
   `id = secretFindingId(path, line, ruleId)` (02); `masked` = first 4 chars + `…` +
   last 2 of the match; the raw match must not be stored on the object or logged
   (unit test asserts the full token appears nowhere in serialized findings/logs).
3. Line numbers: scanner receives per-string metadata; for excerpts, `line` = the
   absolute file line of the match (excerpt start + offset); for narration bodies,
   `line` = null and `path` = the step id.
4. Gate semantics: block findings whose `id` ∈ `config.secretScan.allowlist` move to
   `warned`; remaining block findings throw `SecretGateError` whose message lists ids,
   ruleIds, paths, masked values, and the exact config snippet to allowlist
   (`"secretScan": { "allowlist": ["s…"] }`). `secretScan.enabled: false` skips scanning
   entirely but prints a prominent warning `secretscan-disabled` (allowed for private
   use; documented discouraged).
5. Scan corpus definition (consumed by 25/27): every `CodeExcerpt.text`, every
   `TourStep.body`, `readme.firstParagraph`, and manifest strings that entered narration
   facts. One entry point `scanBundleStrings(bundleParts) → Finding[]` assembles this
   corpus so emit (27) and LLM redaction (25) share it.
6. Redaction helper for LLM path: `redactBlockMatches(text) → text` replacing block-rule
   matches with `[REDACTED:<ruleId>]` (§8.3).

## Acceptance Criteria

- [ ] Table-driven tests: ≥ 1 positive + ≥ 2 negatives per block rule (negatives include near-misses: `AKIA` too short, `ghp_` 35 chars); warn rules likewise; entropy exceptions verified (test file, sha256 context).
- [ ] Hostile fixture flow: `notes.txt` fake AKIA (in an excerpt) → block finding with correct path/line/masked (`AKIA…` + last 2); allowlisting its id turns the run green with a warn entry.
- [ ] `SecretGateError` message contains the copy-pasteable allowlist snippet and no full token.
- [ ] Config-check refactor: issue 03's tests still pass with rules imported from `rules.ts` (no duplicated regex literals remain — grep assertion in test).
- [ ] `redactBlockMatches("key=AKIA…")` replaces only the token span.

## Validation

`pnpm --filter onboard-cli-placeholder test` (secretscan suite + config regression).

## Dependencies

02 (ids), 03 (refactor target). Consumers: 25, 27.

## Non-goals

Scanning the whole repo (only bundle-bound strings — fs-scan name-exclusion covers
sensitive files); git-history scanning; custom user rules (v2).

## Design References

DESIGN §11.4, §8.3, §6.2(3), §11.2 T2, ADR-006.
