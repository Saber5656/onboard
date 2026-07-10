# Title

LLM narration enhancer: facts prompts, budgets, cache, fallback, token report

## Summary

Implement `src/narrate/llm.ts` per DESIGN §8.2–8.5: the `--llm` pass that rewrites each
step's template draft from a facts-only prompt (PROMPT_VERSION = 1), with per-step and
per-run token budgets, pre-send redaction (23), the narration cache, template fallback on
any failure, and the `TokenReport` (including the naive-baseline estimate and the CLI
summary line).

## Context

This is the token-optimization differentiator made concrete (ADR-002): bounded input,
cached, fully reported, and never able to break a generation (fallback path).

## Scope

- `src/narrate/llm.ts` — `enhanceTours({ tours, model, excerpts, config, adapter, cacheFile, logger }) → { tours, tokenReport, warnings }` (model = RepoModel: supplies repoName, locale-independent facts, git facts, module/symbol context, and the analyzed-bytes total for the baseline; excerpts = the registry output for `excerptText` facts).
- `src/narrate/facts.ts` — `NarrationFacts` builder per step (§8.2 field list) + `estimateTokens(text) = ceil(utf8Bytes/4)` (§8.5).
- `src/narrate/cache.ts` — load/save `.onboard/cache/narration-cache.json` (§8.4 format incl. `order: string[]`), corruption-tolerant (bad JSON or schema-invalid entries → start fresh + warning).
- Unit tests with the mock adapter (24).

## Detailed Requirements

1. Prompt exactly §8.2 (system text verbatim, locale interpolated; user message =
   `stableStringify(facts)`); `temperature: 0`; `maxOutputTokens = llm.maxOutputTokensPerStep`.
2. Input budgeting: if `estimateTokens(promptText) > llm.maxInputTokensPerStep`, truncate
   `facts.excerptText` by whole lines from the end until within budget (min 5 lines kept;
   still over → skip step with warning `llm-step-too-large`, keep template).
3. Run budget with concurrency (deterministic reservation): steps are processed in
   bundle order; before dispatching a step, **reserve** its estimated tokens (input
   estimate + `maxOutputTokensPerStep`) against the running committed total; if the
   reservation would exceed `llm.maxTokensPerRun`, stop dispatching — remaining steps
   keep templates, single warning `llm-budget-exhausted` (+ counts). After each
   completion, replace the reservation with provider-reported actuals (estimator when
   absent). Concurrency 2 (simple worker pool); order of results must not affect output
   (steps updated in place by id).
4. Per-step timeout 30 s; 1 retry on `LlmProviderError`/timeout; second failure →
   template fallback.
5. Redaction: `redactBlockMatches` (23) over the serialized facts **before** send;
   the cache key hashes exactly what is sent:
   `sha256(model + ":" + PROMPT_VERSION + ":" + locale + ":" + stableJson(redactedFacts))`
   (§8.4 — locale is in the prompt, so it must be in the key).
6. Output validation (§8.2, shared helper `assertNoRawHtmlLikeMarkup` from 22): reject
   `<`-letter/`</` sequences or > 1200 chars → fallback + warning `llm-output-rejected`.
   Cached bodies are re-validated with the same rule on every hit (a stale/corrupt
   cache entry must not bypass validation); invalid cached entry → treated as miss.
   Accepted output: `bodySource: "llm"`, body = trimmed output.
7. `stepsFallback` increments for every step that ends with a template body after an
   enhancement was *attempted or intended*: provider failure after retry, output
   rejected, prompt too large (`llm-step-too-large`), invalid cache entry with
   subsequent failure, and budget-stopped steps. `stepsEnhanced` + `stepsFromCache` +
   `stepsFallback` = total steps (invariant asserted in tests).
8. `TokenReport` per §5.3: naive baseline = `ceil(Σ FileNode.size over files with
   sha256 !== "" / 4)` (the analyzed-text bytes; binary/oversized files excluded),
   `savingsRatio` clamped ≥ 0. CLI summary line format exactly §8.5. Estimator-derived
   numbers set `estimated: true` in run-report (report field, not bundle).
9. Cache pruning: entries not touched this run are kept; file size > 5 MB → drop oldest
   by `order` (insertion order, §8.4 shape).
10. Without `--llm`: this module is never invoked; `tokenReport: null` in meta.

## Acceptance Criteria

- [ ] Mock happy path: all steps enhanced; bodySource flips to `llm`; captured requests assert the exact system prompt (with locale), stableJson facts payload, `temperature: 0`, and `maxOutputTokens`; streamed deltas assemble correctly and usage items map to the report.
- [ ] Second run with same inputs hits cache 100% (`stepsFromCache = N`, zero adapter calls — assert call count); an `en` run does NOT cache-hit a `ja` run (locale in key).
- [ ] Budget test: `maxTokensPerRun` sized to allow ~2 steps → exactly the first 2 (bundle order) enhanced even with concurrency 2, rest template + `llm-budget-exhausted`.
- [ ] Fallback accounting: one case per class (provider double-failure, output-rejected, step-too-large, budget-stopped, corrupt cache entry) increments `stepsFallback`, and the three counters sum to the step total; `enhanceTours` returns without throwing in every case.
- [ ] Output with `<script>` rejected → fallback + `llm-output-rejected`; a hand-corrupted cache entry containing `<img` is treated as a miss.
- [ ] Facts containing a planted fake AKIA token are redacted in the request payload (mock captures request; assert `[REDACTED:aws-access-key]` present, token absent).
- [ ] TokenReport numbers add up (input+output = Σ per-step actuals) and `savingsRatio` matches a hand-computed value from the fixture's analyzed-bytes total.

## Validation

`pnpm --filter onboard-cli-placeholder test` (llm-narration suite; offline, mock only).
Manual (optional, keyed): one real `--llm` run on mini-express-app by the repo owner
before release (tracked in 41).

## Dependencies

22 (template drafts + facts alignment), 23 (redaction), 24 (adapters + mock).

## Non-goals

Chat (34); prompt experimentation (PROMPT_VERSION bumps are future PRs); streaming
narration to the terminal.

## Design References

DESIGN §8.2–8.5, §5.3 TokenReport, ADR-002, §11.7.
