# Title

CLI skeleton: commands, flags, exit codes, logging, error taxonomy

## Summary

Implement the `onboard` executable per DESIGN §4.1–4.2 and §12: commander-based
`generate` / `serve` / `init` commands with exact flags, the four-code exit contract, an
error taxonomy (`UsageError`/`AnalysisError`/`SecretGateError` with stable `code`s), a
leveled stderr logger with key redaction, and `--json` run-report plumbing. `generate`
and `serve` dispatch to stub runners that later issues (16/27/33) replace.

## Context

Downstream issues plug pipeline/emitter/server implementations into this shell. Locking
flags, exit codes, and logging now means those issues only implement behavior, not UX.

## Scope

- `src/cli/main.ts` (bin entry), `src/cli/generate.ts`, `src/cli/serve.ts`, `src/cli/init.ts` (thin command wiring).
- `src/cli/errors.ts` — error classes + `exitCodeFor(err)`.
- `src/cli/logger.ts` — `createLogger({ level, stream })` with `debug/info/warn/error`, ANSI color (auto-off when `NO_COLOR` set or stream not TTY), and a redaction filter.
- `src/cli/run-report.ts` — accumulator object matching DESIGN §12 run-report shape; written to `.onboard/run-report.json` by generate flows and printed to stdout with `--json`.
- Stub runners: `runGenerate(opts)` / `runServe(opts)` throwing `AnalysisError("not-implemented")` until issues 16/27/33 land.
- Child-process integration tests.

## Detailed Requirements

1. Flags exactly as DESIGN §4.1 (names, defaults, repeatability of `--entry`,
   `--tours` comma list validated against the four `TourKind`s). Unknown flag/command →
   commander error mapped to exit 2 with one-line usage hint. No interactive prompts.
2. Exit codes exactly §4.2: `UsageError`→2, `AnalysisError`→3, `SecretGateError`→4,
   anything else →1 (message + `Please report:` hint). Implement one top-level
   try/catch in `main.ts`; command handlers never call `process.exit` themselves.
3. `--quiet` = errors only; default = info; `--verbose` = debug. All human output to
   **stderr**; stdout is reserved for `--json` (exactly one JSON document, produced with
   `stableStringify`).
4. Redaction filter: at logger construction, snapshot values of
   `ONBOARD_LLM_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`; any occurrence of a
   snapshot value (length ≥ 8) in any log message is replaced with `***`.
5. `init` command calls `writeInitFiles` (issue 03) and prints what happened; config
   exists → exit 2 with its error code.
6. `--version` reads the package.json version (single source).
7. Error `code` values used by this issue: `not-implemented`, `config-*` (from 03),
   `port-in-use` (reserved for 33), `path-not-directory` (validated here: `[path]`
   argument must be an existing directory before dispatch).

## Acceptance Criteria

- [ ] `onboard --help`, `onboard generate --help`, `onboard serve --help` document every §4.1 flag verbatim.
- [ ] Integration tests (spawn the built CLI): `onboard generate /nonexistent` → exit 2 with code `path-not-directory` (path validation is a usage error, not an analysis error; assert stderr contains the code); `onboard generate .` in a temp dir → exit 3 `not-implemented` (until issue 16); `onboard bogus` → exit 2; `onboard init` in temp dir → exit 0 and files exist; second `init` → exit 2.
- [ ] `--json` on a failing stub still emits valid JSON to stdout with `warnings` and error info, and nothing else on stdout (assert stdout parses as JSON).
- [ ] With `ANTHROPIC_API_KEY=sk-test-1234567890` in env, `logger.info("key is sk-test-1234567890")` writes `key is ***`.
- [ ] `NO_COLOR=1` output contains no `\x1b[` sequences.

## Validation

`pnpm --filter onboard-cli-placeholder test` (cli suite runs against `dist/` build);
manual: run the five example invocations from Acceptance and eyeball messages.

## Dependencies

03.

## Non-goals

Real generate/serve behavior (16, 27, 33); shell completions; Windows-specific handling.

## Design References

DESIGN §4.1–4.2, §12, §4.5 (run-report location), §11.2 T5 (log hygiene).
