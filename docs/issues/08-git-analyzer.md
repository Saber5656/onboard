# Title

Git history analyzer: per-file activity, co-change pairs, privacy rules

## Summary

Implement `src/analyze/git/` per DESIGN §6.4: run a capped `git log`, derive per-file
commit counts / last-touched dates / contributor **counts** (identities hashed and
discarded), compute co-change pairs with support+lift thresholds, and capture
`headCommit`, `dirty`, and the HEAD committer date that anchors `meta.generatedAt`.

## Context

Feeds the hotspots tour (§7.4), contributing tour ("where to start"), and the
determinism-relevant `generatedAt` (§5.6 rule 4). Privacy rule T8: no identities in any
output, ever.

## Scope

- `src/analyze/git/index.ts` — `analyzeGit(root, files, config, logger) → { git: GitStats | null, warnings }`.
- `src/analyze/git/log-parse.ts` — parser for the exact log format.
- Unit tests using fixture-git materialized repos (05).

## Detailed Requirements

1. Preconditions: `<root>/.git` exists and `git --version` succeeds; else return
   `git: null` + warning `git-unavailable`. `--no-git` flag short-circuits before this
   stage (pipeline concern, issue 16).
2. Commands (all with `cwd=root`, env `GIT_CONFIG_GLOBAL=/dev/null`,
   `GIT_CONFIG_SYSTEM=/dev/null`, `GIT_TERMINAL_PROMPT=0`, `LC_ALL=C`; every invocation
   prefixed `git -c core.quotepath=false -c core.fsmonitor=false -c core.hooksPath=/dev/null --no-pager`
   — the analyzed repo is untrusted (§11.2 T1): no prompts, hooks, pagers, or fsmonitor.
   Per-command timeout 30 s and stdout cap 64 MB; breach → `git: null` + warning
   `git-command-failed`):
   - `… rev-parse HEAD` → headCommit (no commits yet → `git: null` + `git-empty`).
   - `… status --porcelain` → `dirty` = non-empty output.
   - `… log --no-merges --name-only --format=%H%x00%cI%x00%aE -n 5000 -- .`
   - `… show -s --format=%cI HEAD` → headCommitterDateIso (normalize to UTC `Z`).
3. Parse defensively: stdout decoded as UTF-8 with lossy replacement; entries are
   separated by the `%H%x00%cI%x00%aE` header lines; file lines until next header.
   Each git-reported path is normalized with the **same** rules as fs-scan output
   (repo-relative POSIX, NFC, control-char strip) before FileNode set membership checks
   and before appearing in any output. Skip files not present in the FileNode set.
4. Per-file aggregation: `commits` count, `lastTouchedIso` (max committer date, UTC),
   `commitDatesIso` (all committer dates for the file, UTC, sorted desc, capped at 50 —
   hotspot scoring input, §7.4), `contributorCount` = size of the set of
   `sha256(authorEmail)`. Identity rule (enforceable form): raw author emails/names must
   never be stored in returned objects, warnings, error messages, logs, or module state —
   hash immediately during parsing and keep only hashes for set counting.
   Output ordering: `perFile` sorted by `path` byte order; `commitDatesIso` sorted desc.
5. Co-change: consider only commits touching ≤ 20 analyzed files; for each unordered pair
   increment count; keep pairs `count ≥ 4` and `lift > 2.0`
   (`lift = (pairCount · totalCommits) / (commitsA · commitsB)`); sort by count desc,
   tie lexicographic `(a,b)`; cap 200 (`git-cochange-capped` warning when hit).
6. `commitsAnalyzed` = parsed commit count; when it equals the 5000 cap, warn `git-log-capped`.
7. Determinism: identical materialized fixture ⇒ byte-identical serialized GitStats.

## Acceptance Criteria

- [ ] mini-express-app (12 scripted commits): `src/services/userService.ts` has the max `commits` (= 5) with `commitDatesIso` matching the scripted dates (desc) and `contributorCount` = 2 (two scripted authors, issue 05); pair (`src/db/repo.ts`, `src/services/userService.ts`) present with count = 4 and lift = 2.4; headCommitterDateIso equals the last scripted commit date.
- [ ] `dirty` false after clean materialization; true after the test touches a file (no commit).
- [ ] Privacy sweep: serialized GitStats, all warnings, and captured logs contain none of `fixture@example.com`, `fixture2@example.com`, `Fixture Bot` (grep assertions).
- [ ] Parser-level cap tests (constructed log text, no real repo): exactly 5000 entries → `git-log-capped`; > 200 qualifying pairs → `git-cochange-capped` + truncation to 200.
- [ ] Repo with zero commits (init only) → `git: null` + `git-empty`.
- [ ] Two materializations → identical GitStats JSON (stableStringify equality).

## Validation

`pnpm --filter onboard-cli-placeholder test` (git suite). Reviewer greps test snapshots
for `@` to confirm no email leakage.

## Dependencies

06 (FileNode set), 05 (fixtures).

## Non-goals

Blame-level analysis; rename tracking (`--follow`); shallow-clone deepening; identity
display of any kind (explicit non-goal per §11.9).

## Design References

DESIGN §6.4, §7.4 (consumer), §5.2 GitStats, §5.6 rule 4, §11.2 T8, §11.9.
