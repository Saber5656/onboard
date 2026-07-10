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

1. Builder contract (the one multi-tour kind, 17):
   `buildEntryFlowTours(model, config, strings) → { tours: Tour[], unavailable:
   TourUnavailable | null }` — one tour per surviving CallPath, tour id
   `entry-flow:<entryId>`, output sorted by entryId (§5.6). `unavailable` is set
   (reasonCode `no-traceable-entries`) only when zero tours result **and** the pipeline
   had not already marked the kind unavailable.
2. Step 0 (overview): facts = entry kind, evidence strings (verbatim from 12), file-head
   excerpt of the entry file (`fileHead: true`), symbol name when present.
3. Hop steps (1..H): title from StringTable key `flow.hop` with facts
   `{ callerName, calleeName }`. The hop step's `excerptId` = the call-site excerpt
   (span with `context: true`); its `anchor` = the callee declaration span (where the
   step "lands"). The callee's declaration head is conveyed through **typed body
   facts** — `{ calleeSignature, calleeFile, calleeStartLine }` — rendered by the
   template as inline code + text (no second excerpt, no links to anchors, no raw
   multi-line code in the body; the signature string is already sanitized by 10/14 and
   is covered by the §11.4 narration-body gate).
4. Branch notes: body facts list (`branchNotes: string[]` from the CallPath hop,
   rendered via the `common.branchNotes` fragment); empty list → omitted from facts.
5. Recap step: breadcrumb `entry → hop1 … → hopH` (names only), `truncated` note when
   set (StringTable key `flow.truncated`), pointer to the architecture tour.
6. Entries with `flowEligible: false` or a dropped CallPath simply produce no tour —
   the skip warnings already exist upstream (12 `entries-dropped`, 15
   `flow-too-shallow`); this builder emits **no new warning codes**.
7. Estimated minutes from framework; steps = 2 + H.

## Acceptance Criteria

- [ ] mini-express-app: tour exists for the `src/server.ts` entry; hop steps' anchors land on `src/routes/users.ts`, `src/services/userService.ts`, `src/db/repo.ts` declarations in order (freeze); call-site excerpts have ±3 context lines.
- [ ] Each hop step's body facts carry `calleeSignature`/`calleeFile`/`calleeStartLine` for its callee (asserted on the fixture), and the final hop's facts reference the callee, not any "next step".
- [ ] Overview step carries ≥ 2 evidence strings verbatim from the detector.
- [ ] Recap breadcrumb equals the hop sequence; `truncated: true` case (maxDepth 2 config) shows the truncation fact.
- [ ] Snippet with recursion: recursion branch note propagates into the hop facts.
- [ ] Zero surviving paths → `{ tours: [], unavailable: { reasonCode: "no-traceable-entries" } }` and run-builders (17) flips the merged availability.

## Validation

`pnpm --filter onboard-cli-placeholder test` (entry-flow suite); double-run determinism.

## Dependencies

17 (framework), 15 (callPaths), 12 (entries).

## Non-goals

Multiple alternative paths per entry; framework-route enumeration; narration prose (22).

## Design References

DESIGN §7.3, §6.9–6.10, §5.3 (step shape note), §5.6 (ordering).
