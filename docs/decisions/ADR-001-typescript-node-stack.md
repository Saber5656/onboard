# ADR-001: TypeScript/Node.js stack, pnpm workspaces, npm distribution

Status: Accepted (2026-07-10)

## Context

onboard's v1 deep analysis targets TypeScript/JavaScript codebases, the viewer is a web
app, and implementation will be executed by lower-capability coding agents from written
issues. Candidate stacks: TypeScript/Node, Rust single binary, Go single binary.

## Decision

- Implementation language: **TypeScript** on **Node.js ≥ 20**, pure ESM (`"type": "module"`).
- Repo layout: **pnpm workspaces** — `packages/onboard` (published library + CLI, bin name `onboard`)
  and `packages/viewer` (private Preact SPA built into the CLI package's assets).
- Distribution: **npm** (`npx <pkg>` runs without install). Build via plain `tsc` (no bundler for the CLI).
- Build toolchain TypeScript: `^7.0`; if ecosystem friction blocks (KU-2), documented fallback `~5.9`.
  This does not affect analysis: ts-morph bundles its own compiler (ADR-005).

## Consequences

- TS/JS deep analysis can use the TypeScript compiler ecosystem directly — the strongest
  available option for the primary analysis target.
- Viewer and CLI share one language; implementation agents work in one toolchain.
- Startup latency is worse than a native binary; accepted for v1 (tour generation is not
  latency-sensitive; `serve` is a long-lived process).
- npm package name: `onboard` is taken on npm (verified 2026-07-10, existing package v0.0.10).
  The **bin name stays `onboard`**; the published package name is decoupled and decided at the
  release issue (KU-1 in DESIGN §15.3).

## Alternatives considered

- **Rust/Go single binary**: better startup/distribution, but weak TS type-level analysis
  (tree-sitter syntax only), a two-language codebase (CLI + web viewer), and higher
  implementation risk for delegated agents. Rejected for v1; not precluded later.
