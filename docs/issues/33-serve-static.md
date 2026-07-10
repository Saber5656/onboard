# Title

`onboard serve`: hardened localhost static server and session endpoint

## Summary

Implement `src/server/` per DESIGN §10.1–10.2 and the §11.6 checklist: a `node:http`
server bound to 127.0.0.1 serving only `site/` via sirv, with Host/Origin guards, the
per-process session token, `GET /api/session`, no-CORS policy, body/stream limits
plumbing, `--fresh`/auto-generate behavior, and `--open`.

## Context

serve is the product's only network listener and the security boundary that ADR-003
leans on. Chat (34) mounts onto this server; this issue delivers everything except
`/api/chat` itself.

## Scope

- `src/server/index.ts` — `runServe(opts)` replacing the CLI stub (04): resolve bundle/site (generate when missing; `--fresh` regenerates), start server, print URL, handle SIGINT cleanly.
- `src/server/guards.ts` — request guard chain (§10.1): Host allowlist, `/api/*` Origin check, 403 responses; 404 for anything outside `site/` and `/api/*`.
- `src/server/session.ts` — token generation (`base64url(crypto.randomBytes(32))`), `GET /api/session` response per §10.2 (chat config presence-aware; `enabled: false` + reason without `config.llm` or key).
- Static serving via sirv (`dev: false, etag: true, dotfiles: false`) rooted at the site dir.
- Integration tests over real HTTP (ephemeral port).

## Detailed Requirements

1. Bind exactly `127.0.0.1` (hard-coded; no config/flag to widen in v1). Port from
   `--port` (default 4923); `EADDRINUSE` → `UsageError("port-in-use")` exit 2, no
   auto-increment (§4.1).
2. Guard order (before routing): (a) Host header ∈ {`127.0.0.1:<port>`,
   `localhost:<port>`} else 403 JSON `{code:"bad-host"}`; (b) for `/api/*` paths, when
   `Origin` header present it must equal `http://127.0.0.1:<port>` or
   `http://localhost:<port>` else 403 `{code:"bad-origin"}`. No `Access-Control-*`
   headers on any response. All `/api/*` responses `Cache-Control: no-store`.
3. Path safety: rely on sirv for normalization **and** add a defensive pre-check
   rejecting raw paths containing `..` after URL-decoding (404). `tour-bundle.json`,
   `run-report.json`, `cache/` are outside the served root by §4.5 layout — assert.
4. `/api/session` (§10.2): 200 with token + chat block. Token is per-process; response
   includes `sessionTokensUsed`/`sessionTokenBudget` (wired to real counters in 34;
   zeros here). When serve runs without `config.llm` or without a resolvable key:
   `chat: { enabled: false, reason }` (distinct reasons: `no-config`, `no-api-key`).
5. Bundle resolution: `[path]` arg → repo root; site dir = `<out>/site` (respect
   `--out`); missing → run the full generate flow (16+27 path) with defaults + info log;
   `--fresh` always regenerates. Generate failures propagate their §4.2 exit codes.
6. `--open`: platform opener (`open`/`xdg-open`) spawned detached, failures ignored
   (log debug). URL printed: `http://127.0.0.1:<port>/`.
7. SIGINT: close server, exit 0. In-flight responses get 5 s grace.
8. Logging: one line per request at debug level only (method, path, status — never
   query strings or headers).

## Acceptance Criteria

- [ ] Integration: GET `/` serves the emitted index.html (200, correct content-type); GET `/api/session` returns token (base64url, ≥ 43 chars) and `chat.enabled: false` with reason `no-config` when unconfigured.
- [ ] `Host: evil.example` → 403 `bad-host`; `/api/session` with `Origin: https://evil.example` → 403 `bad-origin`; same requests with correct Host/no Origin → 200.
- [ ] `GET /%2e%2e/tour-bundle.json` and `GET /../run-report.json` → 404; no bundle bytes ever served (assert body).
- [ ] Occupied port → exit 2 `port-in-use`.
- [ ] Missing `.onboard/` in a fixture copy → serve triggers generate then serves (assert site exists after). `--fresh` regenerates: assert via the generate-started log line (output bytes are identical by design, so file-content comparison cannot prove a rerun).
- [ ] SIGINT → clean exit 0 (spawned-process test).

## Validation

`pnpm --filter onboard-cli-placeholder test` (server suite; real sockets on 127.0.0.1);
manual: `onboard serve` on mini-express-app, open the URL, click through the tour.

## Dependencies

04 (CLI), 27 (site/bundle to serve).

## Non-goals

`/api/chat` (34); HTTPS; non-localhost exposure; multi-user auth (§15.1).

## Design References

DESIGN §10.1–10.2, §11.6, §4.1–4.2, §4.5, ADR-003.
