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
- Pages workflow `.github/workflows/pages.yml`: **`workflow_dispatch` only in v1** — build + generate + deploy exactly `.onboard/site/` to GitHub Pages. A commented-out `push: branches: [main]` trigger block plus the owner steps to enable it (and to make Pages public) live in the workflow file header (publication-gate rule: enabling auto-deploy is the owner's explicit act).
- External demo: one mid-size TS OSS repo. The notes must record: repo URL, commit SHA, clone command, why it qualifies (~1–3k TS files), and the analysis policy — **`onboard generate` only; never `npm install` or any script of the target repo** (T1 posture even in dogfood).
- Manual acceptance checklist execution (below) with results recorded in the notes file.
- Real-provider smoke (once, local, owner-run — the agent never handles the key): one `--llm` narration run + one `onboard serve` chat session against the self-tour, confirming tokenReport and meter numbers look sane.
- README update with the published Pages URL is **in this issue's scope** (a docs-only commit), executed only after the owner-approved deployment.
- File follow-up GitHub issues for defects/quality gaps found (each linked from the notes).

## Detailed Requirements

1. **Static viewer checklist** (execute per repo × locale en/ja): all available tours
   open and navigate cleanly; architecture map matches reality (subjective note);
   entry-flow path is truthful (follow it in the code manually and confirm each hop);
   hotspots list is plausible against `git log` spot-checks; contributing commands all
   execute; no rendering glitches in both themes; `file://` open works.
2. **Chat smoke** (once, self-tour, local serve, real provider): the five scripted
   questions with rubric —
   Q1 "What are the main modules of this repo?" (chips include architecture steps; answer names real modules);
   Q2 "How does generate work end to end?" (chips include entry-flow steps);
   Q3 "Where are secrets scanned before publishing?" (answer references the secret gate; no invented file names);
   Q4 "How do I run a single test?" (answer matches the contributing tour's command);
   Q5 "What is the production deployment topology?" (out of scope — answer must be the honest "the tour doesn't cover that" behavior).
   Pass = context chips reference existing steps/excerpts and no fabricated identifiers; each Q graded pass/fail in the notes.
3. **Pages verification**: the deployed artifact contains exactly the files of
   `.onboard/site/` (no bundle, no cache, no run-report); chat UI is absent (no
   `/api/session`); the secret gate passed during the CI generate; deployed HTML
   contains no absolute paths, no hostname, no unmasked secrets (grep checklist in
   notes).
4. Performance record per repo: generation wall time, peak RSS (macOS:
   `/usr/bin/time -l`, Linux: `/usr/bin/time -v` — record which), bundle/site sizes
   (informative vs §13 targets; regressions → follow-up issues, not blockers unless
   caps are violated).
5. ja review: repo owner reads every ja narration key output on the self-tour and marks
   unnatural phrasings; fixes go to issue 22's string tables via follow-ups (KU-7 exit).
6. `docs/research/dogfood-notes.md` uses a fixed template: one row per checklist item
   with columns {item, repo, locale, status pass/fail/n-a, evidence link, notes,
   reviewer}; plus the perf table and the follow-up issue list.
7. v1 closure: when this issue closes, ISSUE_PLAN §1's completion statement is re-read
   and each clause checked off in the closing comment; unresolved items become labeled
   follow-up issues (`v1-followup`).

## Acceptance Criteria

- [ ] `docs/research/dogfood-notes.md` committed using the template: self-tour + external-repo results (incl. repo URL/SHA/rationale), checklist outcomes, chat rubric grades, perf table with measurement method, ja review notes, links to all follow-up issues.
- [ ] Pages workflow merged (dispatch-only, commented push trigger) and one owner-approved deployment succeeded; Pages verification (requirement 3) all green; URL recorded and committed to README within this issue.
- [ ] Real-provider smoke performed by owner: 5/5 rubric rows graded, sane tokenReport (numbers pasted into notes, key never in any artifact).
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
