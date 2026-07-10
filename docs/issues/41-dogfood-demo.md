# Title

Dogfood: onboard's own tour, external-repo demo, and manual acceptance

## Summary

Run the finished product on itself and on one external OSS repository, publish onboard's
own tour via GitHub Pages (owner-gated), perform the manual acceptance walkthrough
(en + ja, all four tours, chat with a real provider once), and file follow-up issues for
everything that reads wrong — closing the v1 plan.

## Context

Scenario S2 (§1.3) says a maintainer publishes a tour; onboard's repo should be the
first maintainer to do it. This issue is the human-quality gate that automated gates
(36–38) cannot cover: narration quality, tour usefulness, ja naturalness (KU-7).

## Scope

- Self-tour: `onboard generate` on this repo; review all tours; commit nothing generated (`.onboard/` stays ignored) — the Pages artifact is built in CI.
- Pages workflow `.github/workflows/pages.yml`: on main push, build + generate + deploy `site/` to GitHub Pages (workflow committed **disabled by default** — `workflow_dispatch` only; enabling scheduled/auto deploy and making Pages public are owner actions documented inside the workflow file header, per the publication-gate rule).
- External demo: run against a cloned mid-size TS OSS repo (pick at implementation; candidates: `fastify/fastify`, `vercel/ms`-class small repo + one ~2k-file repo) — record generation time, bundle size, tour quality notes in `docs/research/dogfood-notes.md`.
- Manual acceptance checklist execution (below) with results recorded in the same notes file.
- Real-provider smoke: one `--llm` narration run + one serve chat session with a real key (owner-provided env; run locally by the owner following a documented script — the agent never handles the key), confirming tokenReport and meter numbers look sane.
- File follow-up GitHub issues for defects/quality gaps found (each linked from the notes).

## Detailed Requirements

1. Acceptance checklist (execute per repo × locale en/ja): all available tours open and
   navigate cleanly; architecture map matches reality (subjective note); entry-flow path
   is truthful (follow it in the code manually and confirm each hop); hotspots list is
   plausible against `git log` spot-checks; contributing commands all execute; no
   rendering glitches in both themes; `file://` open works; chat answers 5 scripted
   questions with correct context chips and honest "not covered" behavior for 1
   out-of-scope question.
2. Performance record: generation wall time + peak RSS + bundle/site sizes for each
   demo repo (informative vs §13 targets; regressions → follow-up issues, not blockers
   unless caps are violated).
3. ja review: repo owner reads every ja narration key output on the self-tour and marks
   unnatural phrasings; fixes go to issue 22's string tables via follow-ups (KU-7 exit).
4. Pages deployment executed once via workflow_dispatch after owner approval;
   the published URL added to README (39 follow-up commit) — **only** after the
   owner's explicit go (publication gate; repo/Pages visibility remains the owner's call).
5. v1 closure: when this issue closes, ISSUE_PLAN §1's completion statement is
   re-read and each clause checked off in the closing comment; unresolved items become
   labeled follow-up issues (`v1-followup`).

## Acceptance Criteria

- [ ] `docs/research/dogfood-notes.md` committed with: self-tour + external-repo results, checklist outcomes, perf table, ja review notes, links to all follow-up issues.
- [ ] Pages workflow merged (dispatch-only) and one owner-approved deployment succeeded; URL recorded.
- [ ] Real-provider smoke performed by owner with sane tokenReport (numbers pasted into notes, key never in any artifact).
- [ ] Every checklist failure has a filed follow-up issue; none are severity "tour is wrong about the code" left open unlabeled.
- [ ] Closing comment maps the ISSUE_PLAN §1 statement clause-by-clause.

## Validation

The committed notes file + follow-up issues + owner sign-off on the closing comment.

## Dependencies

37, 38 (green product), 39 (README), 40 (name; Pages URL uses the final name).

## Non-goals

Marketing/launch posts; multi-repo benchmark suite (v2); fixing found issues inside this
issue (file, don't fix — keep it an acceptance gate).

## Design References

DESIGN §1.3 (S2), §13, §14 (gates this complements), ISSUE_PLAN §1/§6, KU-7.
