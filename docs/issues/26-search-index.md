# Title

BM25 search index builder (generate side)

## Summary

Implement `src/searchindex/` per DESIGN §10.4 (index half) and §5.4: build the
deterministic BM25 index over steps, exported symbols, modules, excerpts, and file paths
with the normative tokenizer, field boosts, and postings format — stored in
`ViewerIndex.search` — plus the shared query-side scoring function that the chat backend
(34) reuses.

## Context

Retrieval quality bounds chat answer quality and cost: this index is what keeps chat
context to ~12k chars instead of a repo dump. Both build and query sides live here so
one module owns tokenizer + scoring symmetry.

## Scope

- `src/searchindex/tokenize.ts` — normative tokenizer (§10.4), in this exact order: insert boundaries on the **original** string (camelCase `fooBar→foo|Bar`, acronym runs `HTTPServer→HTTP|Server`, letter/digit boundaries) → lowercase → split on non-alphanumerics → drop tokens len < 2.
- `src/searchindex/build.ts` — `buildSearchIndex({ tours, symbols, modules, excerpts }) → { index: Bm25Index, warnings }` (§10.4 stored shape: `{ vocab, docs: [{id, type, len}], postings }`; `type ∈ "step"|"symbol"|"module"|"excerpt"` — **no file-path docs in v1**: paths already occur inside symbol/step text, and contextRefs have no file type). Input shapes come from issue 02's schemas (`Tour`, `SymbolRef`, `ModuleInfo`, `CodeExcerpt`).
- `src/searchindex/query.ts` — `queryIndex(index, question, { boostDocIds?, topK }) → { docId, score, type }[]` (BM25 k1=1.2, b=0.75; context boost ×1.3; ties → docId lexicographic).
- Unit tests incl. a known-corpus ranking snapshot.

## Detailed Requirements

1. Document construction with field boosts (§10.4): steps (title tokens ×3, body ×1.5),
   symbols (name ×3, signature ×1), modules (name ×2 with name = basename of path and
   `root` for `"."`, role ×1), excerpts (first 2000 chars ×1). Boost = fractional term
   frequency multiplier at build time; `tf` and `len` are stored as JSON numbers
   (floats allowed — deterministic since inputs and arithmetic order are fixed).
2. Doc ids: `step:<stepId>`, `sym:<file>#<name>`, `mod:<moduleId>`, `x:<excerptId>` —
   stable, `docs` sorted by id (byteOrderCompare), each doc storing its `type`.
3. Stored shape must round-trip through zod (`Bm25IndexSchema` — finalizes the issue-02
   placeholder) and stableStringify; postings keys sorted; posting lists sorted by
   docIdx.
4. Query scoring: standard BM25 (`idf = ln(1 + (N − df + 0.5)/(df + 0.5))`), document
   length normalization with `avgdl` from the index; `boostDocIds` multiplies final
   score ×1.3 (chat's current-tour/step context, §10.4); return topK (default 8) with
   deterministic tie order.
5. Size guard (§13): when excerpts exceed 500, index the first 500 by excerptId asc,
   warning `search-excerpts-capped` (steps/symbols/modules are never capped).
6. No stemming, no stopwords in v1 (documented — determinism and simplicity over
   recall).
7. Secret-gate interaction (documented, not implemented here): every indexed source
   string (step bodies/titles, signatures, excerpt texts) is part of the §11.4 emit
   corpus, so the vocab cannot contain a secret that the gate did not already see —
   issue 36 adds a hostile-fixture regression asserting the fake token is absent from
   the serialized index.

## Acceptance Criteria

- [ ] Tokenizer table tests with frozen expected outputs (boundaries before lowercasing): `getUserById` → `get,user,by,id`; `HTTPServer` → `http,server` (acronym boundary = uppercase-run followed by lowercase splits before the last capital); `HTTPServer2` → `http,server` (digit split; `2` dropped at len < 2); `v2` → dropped entirely; `snake_case_name` → `snake,case,name`.
- [ ] Known-corpus test: 12 hand-written docs, 5 queries with expected top-3 ids each (snapshot with rationale comments).
- [ ] Structure test: a 3-doc tiny corpus serializes to an exact expected JSON (freezes vocab df values, doc types/lens, sorted postings).
- [ ] mini-express-app end-to-end: query "how are users created" ranks the userService symbol or the flow hop step in top-3 (assert membership, not exact order).
- [ ] Cap test: 501 synthetic excerpts → 500 indexed (ids frozen by order), `search-excerpts-capped` warning, no capped doc in postings.
- [ ] Round-trip: build → stableStringify → parse → identical query results.
- [ ] Boost test: same corpus, `boostDocIds` flips a near-tie (constructed case).

## Validation

`pnpm --filter onboard-cli-placeholder test` (searchindex suite); double-run build
equality (byte-identical serialized index).

## Dependencies

02 (schemas: Tour/SymbolRef/ModuleInfo/CodeExcerpt + Bm25Index placeholder), 17
(tours/excerpts produced by builders; runs on their outputs).

## Non-goals

Embeddings (v2); query-side HTTP (34); viewer-side search UI (not in v1 — the index
serves chat; viewer file-tree filtering is simple substring matching in 28).

## Design References

DESIGN §10.4, §5.4, §5.6 (determinism).
