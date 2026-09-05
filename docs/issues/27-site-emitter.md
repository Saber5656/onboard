# Title

Site emitter: bundle serialization, Shiki highlighting, CSP, inline data

## Summary

Implement `src/emit/` per DESIGN §9.1 and §4.5: assemble the final `TourBundle`, run the
secret gate (23), stable-serialize `tour-bundle.json`, pre-render step markdown
(markdown-it, html:false, post-filter) and excerpt highlighting (Shiki dual theme),
compose `site/index.html` with CSP meta + inline JSON payload, copy viewer assets, and
write `run-report.json` — completing the `generate` command end to end.

## Context

The emitter is where the determinism contract (ADR-007), the XSS strategy (§11.5), and
the fail-closed gate (ADR-006) all converge into actual bytes on disk. After this issue,
`onboard generate` produces a working tour (viewer shell may still be the placeholder
until 28–32 land — emitted HTML must not depend on viewer internals beyond the
`app.js`/`app.css` contract).

## Scope

- `src/emit/bundle.ts` — final TourBundle assembly (meta per §5.3: generatedAt rule §5.6-4, configHash, availability, tokenReport, stats) + gate invocation + `tour-bundle.json` write.
- `src/emit/render.ts` — markdown→HTML for step bodies: markdown-it `{html:false, linkify:false}`, rendered **from the token stream with custom renderer rules** (never regex-rewriting serialized HTML): token types mapping to `p em strong code pre ul ol li a blockquote` render normally; `a` renders as a link only when href matches `^#/tour/` (else its text content only, no element); every other token type renders as escaped text; no attributes beyond the validated href are ever emitted (§11.5).
- `src/emit/highlight.ts` — Shiki `codeToHtml` per excerpt: `lang` from **`CodeExcerpt.lang`** (the bundle is the emitter's input), translated via an explicit onboard-lang → Shiki-grammar mapping table (unmapped/unknown → escaped plain `<pre><code>` fallback, also used when Shiki load or highlight throws); dual themes exactly `themes: { light: "github-light", dark: "github-dark" }` (verify against the pinned Shiki version's dual-theme API at implementation — KU-6); highlighter created once per run.
- `src/emit/site.ts` — `index.html` composition (§9.1 item 4: CSP meta exactly as specified; `<script id="onboard-data" type="application/json">` with stableStringify payload; `<html lang>` from locale; title from viewer.title ?? repoName), asset copy, missing-assets failure (exit 1, rebuild hint).
- Wire into `src/cli/generate.ts` (replacing the final stub) + `run-report.json` write + §12 summary output.
- Unit + integration tests.

## Detailed Requirements

1. Site payload = `{ bundle, renderedSteps: Record<stepId, html>, highlightedExcerpts: Record<excerptId, html> }`; the canonical `tour-bundle.json` contains **no HTML** (§9.1 item 2).
2. Serialization: both JSON artifacts via stableStringify (02); payload JSON embedded
   with `<`-escaping (property of stableStringify — assert emitted HTML contains no
   `</script>` inside the data element).
3. `meta.generatedAt`: HEAD committer date when git present and worktree clean; omitted
   (and `dirty: true`) otherwise — never wall clock (§5.6 rule 4).
4. Ordering: tours in fixed kind order, entry-flow subsorted by entryId (§5.6 rule 2);
   excerpts/renderedSteps records key-sorted (stableStringify does this).
5. Secret gate runs over the §11.4-5 corpus **before any file write**; `SecretGateError`
   → nothing written (not even run-report — the report goes to stderr/`--json` stdout
   only in that case; document this in the issue and test it).
6. Shiki setup: `createHighlighter` with the two themes and the §6.2 language set
   (unavailable langs → plaintext); **exact-pin shiki version** in package.json
   (ADR-007) — add the pin in this issue.
7. Output tree exactly §4.5 (`.onboard/tour-bundle.json`, `site/index.html`,
   `site/app.js`, `site/app.css`, `run-report.json`, `cache/` untouched here).
   `--out` respected; output dir created recursively; stale `site/` contents replaced
   atomically (write to temp sibling then rename — determinism of content, not of
   inode timing).
8. Size warning: the UTF-8 byte length of the inline JSON payload (before embedding)
   > 15 MB → warning `emit-site-large` (§9.1).
9. run-report per §12 (timings from 16 + emit timings, counts incl. steps, availability,
   warnings, tokenReport, warn-level findings, capsHit).

## Acceptance Criteria

- [ ] mini-express-app: `onboard generate` exits 0; both artifacts exist; `tour-bundle.json` parses with `TourBundleSchema`; HTML contains the CSP meta verbatim (§9.1) and the data script with escaped `<`.
- [ ] Rendered narration for a step with a planted `<img onerror>` in a module name shows `&lt;img` in HTML (integration with hostile fixture; raw `<img` absent outside the data JSON — precise assertion: parse the emitted HTML, query the data element, strip it, then assert no `<img` in the remainder).
- [ ] Markdown post-filter: external link `[x](https://evil)` renders as text `x` (no anchor); internal `[x](#/tour/architecture/step/2)` renders as anchor.
- [ ] Gate block leaves outputs untouched: run against a **pre-populated** temp `--out` (snapshot every file hash first) with the un-allowlisted hostile fixture → exit 4, every preexisting file byte-identical, no new files, no temp-sibling directories left.
- [ ] Output tree: custom `--out` respected with recursive creation; a stale `site/extra.js` from a previous run is gone after re-emit; a preexisting `cache/narration-cache.json` is byte-identical after emit.
- [ ] `generatedAt` matrix: clean git fixture → field equals HEAD committer date (UTC) and `dirty: false`; dirtied fixture → field absent, `dirty: true`; no-git fixture → field absent.
- [ ] run-report: `.onboard/run-report.json` parses with `stageTimingsMs` (incl. an `emit` key), `counts`, `availability`, `warnings`, `capsHit`; `--json` stdout equals the report content.
- [ ] Size warning: constructed oversized payload (test seam or giant synthetic excerpt set) triggers `emit-site-large` in report + stderr.
- [ ] Dual-theme highlighting present (emitted excerpt HTML contains both theme variable sets per the pinned Shiki dual output), and an unknown-lang excerpt renders as escaped plain `<pre><code>`.
- [ ] Clean fixture: run generate twice → `tour-bundle.json` and all `site/*` files byte-identical (local pre-check of the 37 gate).
- [ ] `js-lib` and `plain-docs` also emit successfully (degraded tour sets).

## Validation

`pnpm --filter onboard-cli-placeholder test` (emit suite) + open the generated
mini-express-app `site/index.html` from `file://` with the placeholder viewer and confirm
the data element parses (`JSON.parse(document.getElementById('onboard-data').textContent)`
in devtools — manual step recorded in PR).

## Dependencies

18, 19, 20, 21 (tours), 22 (bodies), 23 (gate), 26 (index), 16 (pipeline), 02 (serializer).

## Non-goals

Viewer behavior (28–32); serving (33); single-file `--inline-assets` variant (not in v1).

## Design References

DESIGN §9.1, §4.5, §5.3, §5.6, §11.4–11.5, §12, ADR-004, ADR-006, ADR-007, KU-6.
