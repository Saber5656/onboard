# ADR-006: Fail-closed secret gate before any emit

Status: Accepted (2026-07-10)

## Context

Tours embed real source excerpts and are designed to be published (GitHub Pages).
Committed secrets (keys, tokens, private keys) would otherwise flow straight into a
public artifact. Options: warn-only, redact silently, or block emit.

## Decision

- A scanner (rule table in DESIGN §11.4) runs over **every string derived from repo
  content** before it can leave the process: (a) at emit into bundle/site, (b) on
  LLM-bound narration facts.
- **Block-severity findings abort generation with exit code 4** ("fail closed"). The only
  override is an explicit per-finding allowlist entry (`secretScan.allowlist: ["s…id"]`)
  in the config — reviewable, diffable, precise.
- Warn-severity rules (generic assignments, JWT-like, high-entropy) never block; they are
  reported in `run-report.json`.
- Additionally, sensitive files (`.env*`, key material, etc.) are excluded at scan time
  and never read at all (DESIGN §6.2 rule 3).
- Findings are printed masked (first 4 + last 2 chars); the full secret never appears in
  logs or reports.

## Consequences

- Publishing a tour cannot silently leak a matching secret; the failure is loud, local,
  and precise. False positives cost one allowlist line — the right side of the trade for
  a publish-oriented tool.
- Rule list is finite and will miss things (entropy rule is warn-only to avoid noise).
  Residual risk is documented in the privacy statement (DESIGN §11.9): publishing remains
  an explicit user action.

## Alternatives considered

- **Silent redaction at emit**: hides the problem, produces confusing tours, and trains
  users to trust magic. Rejected.
- **Warn-only**: guarantees an eventual public leak given enough users. Rejected.
