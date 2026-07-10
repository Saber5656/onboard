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

- `src/secretscan/rules.ts` — exported rule types and table:
  `type SecretRule = { ruleId: string; severity: "block" | "warn"; kind: "regex"; patterns: RegExp[] } | { ruleId: "high-entropy"; severity: "warn"; kind: "entropy" }`
  (`github-token` carries its two patterns in one rule; all regexes are case-sensitive with no global flag surprises — create fresh or use `matchAll`).
- `src/secretscan/scan.ts` — `scanStrings(items: ScanPart[]) → Finding[]` where
  `ScanPart = { text: string; path: string; startLine: number | null; isTest: boolean }`,
  plus `shannonEntropy(s)`. Match line = `startLine + newline offset within text` when
  `startLine` is a number, else null.
- `src/secretscan/gate.ts` — `enforceGate(findings, allowlist) → { blocked: Finding[], warned: Finding[] }`; blocked non-empty → `SecretGateError` (exit 4). There is **no global disable** (ADR-006; the config has no `secretScan.enabled` field).
- `src/secretscan/corpus.ts` — `collectScanParts(bundleParts) → ScanPart[]` assembling the normative §11.4 corpus: excerpt texts (startLine from registry, isTest from FileNode), step bodies **and titles** (path = stepId, startLine null), `readme.firstParagraph` (path = the readme file), manifest script strings (path = `package.json`), symbol `signature`/`jsdocSummary` strings (path = symbol file, startLine = symbol line).
- Refactor: `src/config/load.ts` key-in-config check now imports the block rules from `rules.ts` (removing the duplicated regex list from issue 03).
- Unit tests (table-driven positives/negatives per rule).

## Detailed Requirements

1. Rules exactly as the DESIGN §11.4 tables (ids, patterns, severities). Entropy rule:
   candidate tokens = `[A-Za-z0-9+/=_-]{32,}` sequences; compute Shannon entropy over
   chars; > 4.0 bits ⇒ finding, **except** when the part's `isTest` flag is true or
   the surrounding 40 chars contain `sha256|integrity|hash` (case-insensitive).
2. `Finding = { id, ruleId, severity: "block"|"warn", path, line: number | null, masked }`;
   `id = secretFindingId(path, line, ruleId)` (02 — null line hashes as `"-"`);
   `masked` = first 4 chars + `…` + last 2 of the match; the raw match must not be
   stored on the object or logged (unit test asserts the full token appears nowhere in
   serialized findings/logs).
3. Gate semantics: block findings whose `id` ∈ `config.secretScan.allowlist` move to
   `warned`; remaining block findings throw `SecretGateError` whose message lists ids,
   ruleIds, paths, masked values, and a copy-pasteable config snippet containing the
   **actual finding ids** (e.g. `"secretScan": { "allowlist": ["s1a2b3c4d5e6"] }` —
   never the masked secret).
4. The corpus assembly (`collectScanParts`) is the single shared entry point: emit (27)
   gates on it; the LLM path (25) redacts with the same rules.
5. Redaction helper for LLM path: `redactBlockMatches(text) → text` replacing block-rule
   matches with `[REDACTED:<ruleId>]` (§8.3).

## Acceptance Criteria

- [ ] Table-driven tests: ≥ 1 positive + ≥ 2 negatives per block rule (negatives include near-misses: `AKIA` too short, `ghp_` 35 chars; both `github-token` patterns covered); warn rules likewise; entropy exceptions verified (isTest part, sha256 context).
- [ ] Hostile fixture flow: `notes.txt` fake AKIA (in an excerpt) → block finding with the correct absolute file line (excerpt startLine + offset) and mask; allowlisting its id turns the run green with a warn entry.
- [ ] `SecretGateError` message contains the copy-pasteable allowlist snippet built from real finding ids and no full token.
- [ ] Narration-body finding (planted in a step title) gets a stable id across two runs (null-line `"-"` hashing).
- [ ] Config-check refactor: issue 03's tests still pass with rules imported from `rules.ts` (no duplicated regex literals remain — grep assertion in test).
- [ ] `redactBlockMatches` on a string containing a full fake `AKIA` + 16-char token replaces exactly the token span with `[REDACTED:aws-access-key]` (assert the fake token absent, surrounding text intact).

## Validation

`pnpm --filter onboard-cli-placeholder test` (secretscan suite + config regression).

## Dependencies

02 (ids), 03 (refactor target). Consumers: 25, 27.

## Non-goals

Scanning the whole repo (only bundle-bound strings — fs-scan name-exclusion covers
sensitive files); git-history scanning; custom user rules (v2); any global disable
switch (explicitly rejected, ADR-006).

## Design References

DESIGN §11.4, §8.3, §6.2(3), §11.2 T2, ADR-006.
