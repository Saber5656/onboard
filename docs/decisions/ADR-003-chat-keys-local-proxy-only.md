# ADR-003: Chat runs only through the local serve proxy — API keys never reach the browser

Status: Accepted (2026-07-10, owner-confirmed)

## Context

v1 includes Q&A chat in the viewer, but the viewer is also published as a static site
(GitHub Pages). Chat needs an LLM key. Options: (A) local `onboard serve` proxy only,
(B) A + browser BYO-key for static deployments, (C) browser direct calls only.

## Decision

Option **A**: chat is available only when the tour is served by `onboard serve` on
127.0.0.1. The key comes from environment variables, stays inside the CLI process, and is
never sent to, stored in, or entered into the browser. Static deployments (Pages,
`file://`) detect the absence of `/api/session` and hide the chat UI with an explanatory
hint (DESIGN §9.2, §10.2).

## Consequences

- Simplest credible security story for a public OSS tool: no "paste your API key into a
  web page" pattern, no key-in-localStorage risk, no XSS-to-key-theft escalation.
- Published static tours have no chat. Accepted trade-off; revisit as v2 item
  (browser BYO-key, DESIGN §15.2 #4) only with a dedicated threat-model review.
- `serve` becomes a security surface and is hardened accordingly (DESIGN §10.1–10.2,
  §11.6): localhost bind, Host/Origin checks, per-process session token, no CORS.

## Alternatives considered

- **B (BYO-key opt-in)**: maximum reach, but normalizes a dangerous UX and raises the
  viewer's XSS stakes from "defacement" to "credential theft". Deferred, not mixed into v1.
- **C (browser-direct only)**: minimal code, unacceptable key exposure. Rejected.
