# Title

Entry-point flow tour builder

## Summary

Implement `src/tours/entry-flow.ts` per DESIGN §7.3: for each flow-eligible entry point
with a selected CallPath (≥ 2 hops), build a tour `entry-flow:<entryId>` — entry
overview step (evidence + file-head excerpt), one step per hop (call-site excerpt +
callee head, branch notes), and a recap step with the hop breadcrumb and honest
truncation note.

## Context

The flow tours are the product's deepest payoff ("how does a request actually travel").
They must stay honest: branch notes and `truncated` flags surface what the linear path
skips (KU-5).

## Scope

- `src/tours/entry-flow.ts` — the builder.
- Unit tests on mini-express-app + snippet projects.

## Detailed Requirements

1. One tour per entry in `callPaths` order (entries sorted by score desc from 12; tour id
   `entry-flow:<entryId>`; §5.6 ordering rule: entry-flow tours sorted by entryId in the
   final bundle — assembly order concern for 17/27, but this builder must emit
   deterministic per-entry output).
2. Step 0 (overview): facts = entry kind, evidence strings (verbatim from 12), file-head
   excerpt of the entry file (`fileHead: true`), symbol name when present.
3. Hop steps (1..H): title from StringTable key `flow.hop` with facts
   `{ callerName, calleeName }` (rendered like `createUser → repo.insert` — exact
   formatting lives in the string table, not the builder); excerpt = call-site span with
   `context: true`; a second excerpt of the callee declaration head
   (min(10 lines, declaration length)) is attached via the step body facts —
   **model note**: TourStep has a single `excerptId`; the callee-head excerpt is
   referenced from the body markdown as an internal link to the *next* step's anchor
   instead of a second excerpt (keeps the schema of §5.3 unchanged). The hop step's
   `excerptId` = call-site excerpt; its `anchor` = callee declaration span (where the
   step "lands"). Follow this split exactly.
4. Branch notes: rendered into the body facts as a list (`also branches to: a (users.ts), b (health.ts)`); empty list → omitted from facts.
5. Recap step: breadcrumb `entry → hop1 … → hopH` (names only), `truncated` note when
   set (StringTable key `flow.truncated`), pointer to architecture tour.
6. Entries with `flowEligible: false` or missing/short CallPath: no tour; the pipeline
   availability (16) already reports entry-flow unavailable only when **zero** tours
   result; partial cases (some entries usable) proceed with warnings `flow-entry-skipped`.
7. Estimated minutes from framework; steps = 2 + H.

## Acceptance Criteria

- [ ] mini-express-app: tour exists for `src/server.ts` entry; hop steps' anchors land on `routes/users.ts`, `services/userService.ts`, `db/repo.ts` declarations in order (freeze); call-site excerpts have ±3 context lines.
- [ ] Overview step carries ≥ 2 evidence strings verbatim from the detector.
- [ ] Recap breadcrumb equals the hop sequence; `truncated: true` case (maxDepth 2 config) shows the truncation fact.
- [ ] Snippet with recursion: recursion branch note propagates into the hop facts.
- [ ] Zero eligible entries (plain-docs) → builder returns nothing and pipeline availability already says unavailable (integration assert via 16's availability, not re-implemented here).

## Validation

`pnpm --filter onboard-cli-placeholder test` (entry-flow suite); double-run determinism.

## Dependencies

17 (framework), 15 (callPaths), 12 (entries).

## Non-goals

Multiple alternative paths per entry; framework-route enumeration; narration prose (22).

## Design References

DESIGN §7.3, §6.9–6.10, §5.3 (step shape note), §5.6 (ordering).
