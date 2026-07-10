# ADR-004: Self-contained static viewer — IIFE bundle, inline data, generate-time highlighting

Status: Accepted (2026-07-10)

## Context

The tour must open (1) from `file://` with zero setup, (2) via `onboard serve`, and
(3) on static hosts like GitHub Pages — identically, with no external network access.

## Decision

- Viewer = Preact SPA compiled to a **single classic-script IIFE bundle** (`app.js`) +
  one stylesheet. No ESM `<script type="module">` (blocked by CORS on `file://`), no
  code-splitting, no CDN, no web fonts, no external requests (CSP `default-src 'none'`
  with `'self'` carve-outs; DESIGN §9.1).
- Tour data is **inlined** into `index.html` as
  `<script id="onboard-data" type="application/json">` with every `<` escaped as `\u003c`
  (no fetch needed on `file://`; script-tag breakout impossible).
- Syntax highlighting and markdown→HTML run **at generate time** (Shiki, markdown-it) —
  the browser never parses repo content; the viewer only mounts pre-rendered, sanitized
  HTML through two audited sinks (DESIGN §11.5).
- The emitted HTML is byte-identical whether viewed statically or via serve; serve-mode
  features activate through runtime detection (`GET /api/session`), never through emitted
  variants (preserves the determinism contract, DESIGN §5.6).

## Consequences

- One artifact serves all three scenarios; publishing = copying a directory.
- Generate-time rendering keeps the viewer small, fast, and XSS-minimal, at the cost of a
  larger HTML payload (budget: warn > 15 MB, DESIGN §9.1).
- Locale toggle swaps UI strings only; narration language is fixed at generation.

## Alternatives considered

- Runtime highlighting (smaller payload, but ships a highlighter + parses untrusted
  content in-browser). Rejected.
- Separate `data.json` fetched at boot (cleaner, but dead on `file://`). Rejected.
- SSR/MPA static site generator (heavier toolchain, no benefit for an SPA-sized app). Rejected.
