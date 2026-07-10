# Title

Config schema, defaults, loader, and `onboard init` file writer

## Summary

Implement `onboard.config.json` handling per DESIGN §4.3: strict zod schema with the
normative defaults, file discovery/loading, CLI-flag merge precedence, API-key-in-config
rejection, plus the `init` writer (default config file + idempotent `.gitignore` append).
Ship a generated JSON Schema for editor completion.

## Context

The effective config feeds every pipeline stage and is hashed into `meta.configHash`
(determinism). Keys must never live in config (ADR-003/§4.3); init is the first-run UX.

## Scope

- `packages/onboard/src/config/schema.ts` — zod schema + defaults exactly as the §4.3 listing (every field, every default value).
- `packages/onboard/src/config/load.ts` — `loadConfig({ root, configFlag, cliOverrides }) → { config, configHash, sourcePath | null }`; errors thrown as `UsageError` from `src/util/errors.ts` (issue 02).
- `packages/onboard/src/config/init.ts` — `writeInitFiles(root)` and `ensureGitignoreEntry(root)` (separately exported, both used by the CLI `init` command that issue 04 wires; `init` operates on `process.cwd()`).
- `packages/onboard/schemas/onboard.config.schema.json` — generated from zod (committed artifact with a test that regenerates and diffs).
- Unit tests.

## Detailed Requirements

1. Strict parsing: unknown keys anywhere → `UsageError` code `config-unknown-key` listing
   the full key paths (e.g. `llm.apiKey`). zod `.strict()` on every object level.
2. Discovery: `--config <file>` wins (missing file → `config-not-found`); else
   `<root>/onboard.config.json` if present; else pure defaults. JSON only (no JSONC).
   A top-level `$schema` string key is **stripped before validation** (never part of the
   effective config or `configHash`); `writeInitFiles` writes it for editor support.
3. Merge precedence: CLI flags > config file > defaults. `cliOverrides` shape is exactly
   `{ tours?: TourKind[], locale?: "en"|"ja", entryPoints?: ConfigEntryPoint[],
   llmRequested?: boolean }` — `llmRequested` is passed through untouched (the generate
   command, issue 04/16, enforces "requires `config.llm`" per §4.1; there is no
   `llm.enabled` config field). Merge semantics: arrays (`tours`, `entryPoints`,
   `include`, `exclude`, `secretScan.allowlist`) **replace**, never concatenate; objects
   `flow/hotspots/excerpt/llm/chat/secretScan/viewer` merge per-field; `null` is invalid
   everywhere except `llm.baseUrl` and `viewer.title` (strict schema — absence is
   expressed by omission).
4. Cross-field rules: `llm.baseUrl` **required non-null** when
   `provider = "openai-compat"` (`config-baseurl-required`) and must be an http(s) URL;
   for `provider = "anthropic"` it must be null or omitted — any non-null value →
   `config-baseurl-forbidden`. Every **effective** `entryPoints[].file` (from config
   file or `--entry`) is validated to exist under root via `safeJoin` — nonexistent →
   `config-entry-missing`.
5. Key-in-config rejection: scan all string values under `llm` against these regexes
   (copied verbatim from DESIGN §11.4 block rules — issue 23 later refactors both call
   sites to one shared `secretscan/rules.ts`):
   `-----BEGIN( RSA| EC| OPENSSH| DSA| PGP)? PRIVATE KEY-----`, `\bAKIA[0-9A-Z]{16}\b`,
   `\b(ghp|gho|ghu|ghs|ghr)_[A-Za-z0-9]{36,}\b`, `github_pat_[A-Za-z0-9_]{60,}`,
   `\bxox[baprs]-[A-Za-z0-9-]{10,}\b`, `\bsk_live_[A-Za-z0-9]{16,}\b`,
   `\bAIza[0-9A-Za-z_\-]{35}\b`, `\bnpm_[A-Za-z0-9]{36}\b`. Match → `config-contains-secret`
   (message never echoes the value).
6. `configHash` = `sha256(stableStringify(effectiveConfig))` — computed after defaults +
   merges so identical effective configs hash identically regardless of source.
7. `writeInitFiles`: writes `onboard.config.json` (defaults, `$schema` pointing at the
   package schema path) — refuses to overwrite an existing file (`init-config-exists`);
   appends `.onboard/` to `.gitignore` only when not already present (creates the file if
   absent); returns a summary `{ wroteConfig, updatedGitignore }`.

## Acceptance Criteria

- [ ] Defaults object deep-equals the DESIGN §4.3 listing (single table-driven test).
- [ ] Unknown key `llm.apiKey` → exit-2-class `UsageError` with path in message.
- [ ] Cross-field tests: `openai-compat` without `baseUrl` → `config-baseurl-required`; `anthropic` with non-null `baseUrl` → `config-baseurl-forbidden`; `anthropic` with `baseUrl: null` → valid; config-file `entryPoints` pointing at a missing file or outside root → `config-entry-missing`; `--config` pointing at a missing file → `config-not-found`.
- [ ] `$schema` key present in the file: parses fine, and `configHash` equals the same config without `$schema`.
- [ ] A `ghp_…` (36 alphanumerics) planted in `llm.model` → `config-contains-secret`; message does not contain the token.
- [ ] Same effective config from (file) and (defaults + equivalent flags) produces identical `configHash`.
- [ ] `writeInitFiles` twice: second run throws `init-config-exists` (config untouched); calling the `.gitignore` append path again leaves the file byte-identical (idempotent).
- [ ] Generated JSON Schema validates the default config and rejects an unknown-key sample (test uses a JSON-Schema validator only in tests, not runtime).

## Validation

`pnpm --filter onboard-cli-placeholder test` (config suite); mutation spot-check: flip
one default in schema.ts and confirm the table-driven defaults test fails.

## Dependencies

02.

## Non-goals

CLI arg parsing itself (04); env-var key resolution (24); the shared rules refactor (23).

## Design References

DESIGN §4.3–4.4, §5.6 (configHash), §11.4 (rule regexes), ADR-003.
