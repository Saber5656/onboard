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

- `src/narrate/llm.ts` — `enhanceTours(tours, model, config, adapter, cacheFile, logger) → { tours, tokenReport, warnings }`.
- `src/narrate/facts.ts` — `NarrationFacts` builder per step (§8.2 field list) + `estimateTokens(text) = ceil(utf8Bytes/4)` (§8.5).
- `src/narrate/cache.ts` — load/save `.onboard/cache/narration-cache.json` (§8.4 format), corruption-tolerant (bad JSON → start fresh + warning).
- Unit tests with the mock adapter (24).

## Detailed Requirements

1. Prompt exactly §8.2 (system text verbatim, locale interpolated; user message =
   `stableStringify(facts)`); `temperature: 0`; `maxOutputTokens = llm.maxOutputTokensPerStep`.
2. Input budgeting: if `estimateTokens(promptText) > llm.maxInputTokensPerStep`, truncate
   `facts.excerptText` by whole lines from the end until within budget (min 5 lines kept;
   still over → skip step with warning `llm-step-too-large`, keep template).
3. Run budget: running total of (provider-reported input+output; estimator when absent)
   — before each call, if total + step estimate > `llm.maxTokensPerRun`, stop enhancing:
   remaining steps keep templates, single warning `llm-budget-exhausted` (+ counts).
4. Concurrency 2 (simple worker pool); per-step timeout 30 s; 1 retry on
   `LlmProviderError`/timeout; second failure → fallback to template
   (`bodySource: "template"`, `stepsFallback++`). Order of results must not affect
   output (steps updated in place by id).
5. Redaction: `redactBlockMatches` (23) over the serialized facts **before** send;
   redacted prompt is what gets cached-keyed (cache key = sha256(model + ":" +
   PROMPT_VERSION + ":" + redactedFactsJson), §8.4).
6. Output validation (§8.2): reject when output contains `<` followed by ASCII letter or
   `/`, or > 1200 chars → fallback + warning `llm-output-rejected`. Accepted output:
   `bodySource: "llm"`, body = trimmed output.
7. `TokenReport` per §5.3: narration counts (enhanced/fromCache/fallback), naive baseline
   = `ceil(Σ bytes of all read text FileNodes / 4)`, `savingsRatio` clamped ≥ 0. CLI
   summary line format exactly §8.5. Estimator-derived numbers set `estimated: true`
   in run-report (report field, not bundle).
8. Cache pruning: entries not touched this run are kept (cheap re-runs after partial
   edits); file size > 5 MB → drop oldest by insertion order (documented; cache is
   an object — maintain `order: string[]` alongside entries).
9. Without `--llm`: this module is never invoked; `tokenReport: null` in meta.

## Acceptance Criteria

- [ ] Mock happy path: all steps enhanced; bodySource flips to `llm`; second run with same inputs hits cache 100% (`stepsFromCache = N`, zero adapter calls — assert call count).
- [ ] Budget test: `maxTokensPerRun` sized to allow ~2 steps → exactly the first 2 (deterministic order) enhanced, rest template + `llm-budget-exhausted`.
- [ ] Failure injection (mock error twice) → fallback for that step only; run exit code 0.
- [ ] Output with `<script>` rejected → fallback + `llm-output-rejected`.
- [ ] Facts containing a planted fake AKIA token are redacted in the request payload (mock captures request; assert `[REDACTED:aws-access-key]` present, token absent).
- [ ] TokenReport numbers add up (input+output = Σ per-step) and `savingsRatio` matches hand-computed value for the fixture.

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
