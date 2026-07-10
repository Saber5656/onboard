# Title

README, SECURITY.md, CONTRIBUTING.md, and the privacy statement

## Summary

Write the public-facing repository documentation: a README that demonstrates the
90-second value path (`npx … generate && … serve`) and states the two differentiation
claims with their proof mechanisms, SECURITY.md (threat-model summary, report channel,
supported versions, §11.9 privacy statement), CONTRIBUTING.md (dev setup mirroring the
contributing tour's own advice), and doc-level consistency with DESIGN.md.

## Context

This repo is built to go public (user rule: publication itself is a separate gated
step). Docs must be accurate against the implemented v1 — hence late in the plan — and
honest: claims limited to what gates 36/37 actually prove.

## Scope

- `README.md` (rewrite; keep the existing Japanese one-liner as a translated tagline note): what/why, 90-second quickstart, tour types table, determinism claim + how to verify (`generate` twice, diff), token claim + where to see it (tokenReport, chat meter), LLM setup (env keys, provider config), publishing to Pages (chat auto-disabled note), config reference link, security/privacy summary link, v2 roadmap pointer to ISSUE_PLAN §7.
- `SECURITY.md`: report channel (GitHub private vulnerability reporting), response expectation, supported-versions table (v1 line), threat-model summary table (from §11.2, linked to DESIGN), §11.9 privacy statement verbatim, secret-gate behavior and its limits (honest residual-risk wording from ADR-006).
- `CONTRIBUTING.md`: pnpm setup, build/test/lint commands, fixture materialization note, golden-update flow (37), determinism rules for contributors (§5.6 checklist — the "no Date.now()" list), PR expectations (CI gates).
- Cross-checks: every command in the docs is executed once during validation; marketing-adjacent prior-art claims re-verified per research doc's caveat or removed.

## Detailed Requirements

1. README quickstart targets the released package name — until KU-1 resolves, use a
   clearly marked placeholder (`npx <package-name>` with a callout) explicitly excluded
   from command-transcript validation; the transcript instead runs the local
   equivalents (`pnpm exec onboard generate` etc.). Issue 40 does the mechanical
   replace.
2. Claims discipline: determinism wording = "byte-identical output for the same commit,
   config, and version — enforced in CI"; token wording = "narration/chat consume only
   pre-digested context; every run reports actual tokens vs. a full-dump baseline". No
   superlatives, no unverified competitor comparisons (research §1 caveat).
3. English as primary; a short 日本語 section at the bottom of README (tagline +
   quickstart) since the tool ships ja narration — keep it ≤ 20 lines.
4. SECURITY.md includes the **complete** §11.6 hardening checklist as a "what serve
   does / doesn't do" user-facing table — every item: 127.0.0.1 bind, Host allowlist,
   Origin checks on /api/*, per-process session token, no CORS headers, no-store on
   /api/*, 64 KB body limit, 2-stream limit, no directory listing, dotfiles denied,
   only `site/` served, no absolute paths in responses, port conflict = exit (no
   fallback). Plus: no telemetry, keys never reach the browser. The §11.9 privacy
   statement appears **verbatim in both README and SECURITY.md** (§11.9 requires both).
5. Internal links checked by `packages/onboard/test/docs/internal-links.test.ts`
   (runs inside the existing `unit` CI job — no new CI job; external links are not
   checked in CI to avoid flakes).

## Acceptance Criteria

- [ ] Every executable command in README/CONTRIBUTING (placeholder-name commands excluded and marked) executed successfully during review via local equivalents (evidence: transcript in PR).
- [ ] README states both claims with their verification paths, contains the KU-1 placeholder callout, and carries the §11.9 privacy statement.
- [ ] SECURITY.md contains: report channel, versions table, privacy statement (§11.9 verbatim), the complete §11.6 table, secret-gate limits paragraph.
- [ ] `internal-links.test.ts` passes; no absolute file paths or user-specific strings anywhere (grep for `/Users/`, `Saber5656` outside repo URL).
- [ ] ja README section reviewed by the repo owner (native check — request in PR).

## Validation

`pnpm test` (link-check), command transcript, owner review of the ja section.

## Dependencies

27 (generate behavior), 33 (serve behavior), 36 (security claims are test-proven),
37 (verification flows exist).

## Non-goals

Docs site (v2); API/library-usage docs beyond a minimal "programmatic use" pointer;
release/publish instructions (40); localized full README.

## Design References

DESIGN §1.2–1.4, §11.6, §11.9, ADR-002, ADR-006, research/prior-art.md, KU-1.
