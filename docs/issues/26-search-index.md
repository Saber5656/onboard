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

- `src/searchindex/tokenize.ts` — normative tokenizer (§10.4): lowercase → split on non-alphanumerics → camelCase split (`fooBar` → `foo`,`bar`; also split on digit/letter boundaries) → drop tokens len < 2.
- `src/searchindex/build.ts` — `buildSearchIndex({ tours, symbols, modules, excerpts, files }) → Bm25Index` (§10.4 stored shape: `{ vocab, docs, postings }`).
- `src/searchindex/query.ts` — `queryIndex(index, question, { boostDocIds?, topK }) → { docId, score, type }[]` (BM25 k1=1.2, b=0.75; context boost ×1.3; ties → docId lexicographic).
- Unit tests incl. a known-corpus ranking snapshot.

## Detailed Requirements

1. Document construction with field boosts (§10.4): steps (title tokens ×3, body ×1.5),
   symbols (name ×3, signature ×1), modules (name/path ×2, role ×1), excerpts (first
   2000 chars ×1), file paths (path segments ×2). Boost = token frequency multiplier at
   build time (document as integer-weighted tf; k1/b applied at query).
2. Doc ids: `step:<stepId>`, `sym:<file>#<name>`, `mod:<moduleId>`, `x:<excerptId>`,
   `file:<path>` — stable, sorted in `docs` array by id.
3. Stored shape must round-trip through zod (`Bm25IndexSchema` — add to model with this
   issue, coordinate §5.4) and stableStringify; postings keys sorted; document lengths
   stored as integers.
4. Query scoring: standard BM25 (`idf = ln(1 + (N − df + 0.5)/(df + 0.5))`), document
   length normalization with `avgdl` from the index; `boostDocIds` multiplies final score
   ×1.3 (chat's current-tour/step context, §10.4); return topK (default 8) with
   deterministic tie order.
5. Size guard: skip excerpt docs beyond 500 excerpts (largest bundles), warning
   `search-excerpts-capped` — steps/symbols/modules/files are never capped.
6. No stemming, no stopwords in v1 (documented — determinism and simplicity over recall).

## Acceptance Criteria

- [ ] Tokenizer table tests with frozen expected outputs: `getUserById` → `get,user,by,id`; `HTTPServer` → `http,server` (acronym boundary = uppercase-run followed by lowercase splits before the last capital); `HTTPServer2` → `http,server` (digit split; `2` dropped at len < 2); `v2` → dropped entirely; `snake_case_name` → `snake,case,name`.
- [ ] Known-corpus test: 12 hand-written docs, 5 queries with expected top-3 ids each (snapshot with rationale comments).
- [ ] mini-express-app end-to-end: query "how are users created" ranks the userService symbol or the flow hop step in top-3 (assert membership, not exact order).
- [ ] Round-trip: build → stableStringify → parse → identical query results.
- [ ] Boost test: same corpus, `boostDocIds` flips a near-tie (constructed case).

## Validation

`pnpm --filter onboard-cli-placeholder test` (searchindex suite); double-run build
equality (byte-identical serialized index).

## Dependencies

17 (tours/excerpts shapes; runs on builder outputs).

## Non-goals

Embeddings (v2); query-side HTTP (34); viewer-side search UI (not in v1 — the index
serves chat; viewer file-tree filtering is simple substring matching in 28).

## Design References

DESIGN §10.4, §5.4, §5.6 (determinism).
