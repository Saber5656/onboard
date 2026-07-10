# ADR-002: Deterministic core with optional, metered LLM enhancement

Status: Accepted (2026-07-10)

## Context

The product thesis (owner decision, 2026-07-10): "anything can be produced by letting an
agent burn tokens — differentiation must come from deterministic generation or from
measurable token optimization." Tour generation could be (a) fully deterministic,
(b) deterministic skeleton + LLM narration, or (c) LLM-driven with caching.

## Decision

- The pipeline (analysis → tour structure → template narration → site) is **fully
  deterministic and LLM-free by default**. No API key, no network, byte-identical output
  (DESIGN §5.6).
- The LLM is an **optional enhancement layer** (`--llm` narration; serve-time chat) that
  receives only pre-digested facts/skeletons, never raw repo dumps; consumption is capped,
  cached, and reported (`tokenReport`, DESIGN §8).
- Every feature must preserve: deterministic mode remains fully functional, and any LLM
  usage is measured and visible.

## Consequences

- CI-safe, reproducible, key-free default → the determinism differentiation axis holds.
- The token-optimization axis becomes *demonstrable*: the bundle reports actual tokens vs.
  a naive full-repo-dump baseline.
- Template narration quality ceiling is lower than LLM prose; accepted — LLM mode exists
  exactly for that, with fallback to templates on any failure.
- Two narration paths must be maintained (template + LLM rewrite of the template draft);
  the shared typed-facts input keeps them aligned (DESIGN §8.1–8.2).

## Alternatives considered

- **LLM-first with caching**: highest prose quality, but breaks the key-free default,
  weakens reproducibility, and blurs the line against "just ask an agent". Rejected.
- **Deterministic-only v1**: simplest, but chat (a confirmed v1 requirement) needs an LLM
  anyway, so the adapter layer must exist regardless. Rejected as artificial.
