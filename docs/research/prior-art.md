# Prior Art & Positioning Research

Date: 2026-07-10. Method: comparison from maintained knowledge of these tools (training
data through 2026-01) plus registry checks run locally today. **Re-verify feature claims
against current upstream docs before using them in public marketing copy** (tracked in
issue 39 acceptance criteria).

## 1. Landscape

| Tool | What it is | Generation | Consumption | Deterministic? | Token cost | Publishable artifact |
|---|---|---|---|---|---|---|
| **CodeTour** (VS Code extension, MS) | Guided step-through tours of a repo, steps anchored to file+line | **Manual** authoring | Interactive, inside VS Code only | n/a (hand-written) | none | `.tours/*.tour` JSON in-repo |
| **Swimm** | Commercial doc platform coupling docs to code, auto-syncs on change | Manual docs + auto-sync engine | Web/IDE, SaaS | no | vendor-side | SaaS-hosted |
| **repomix** | Packs a repo into one LLM-friendly text/XML file | Deterministic pack (no LLM) | LLM context input, not human-facing | yes | shifts cost to *every* downstream prompt (full dump) | single packed file |
| **Aider repo map** | tree-sitter symbol map ranked (PageRank-style) to fit a token budget | Deterministic map | LLM context for aider sessions | yes | optimized per-session | internal (not a standalone artifact) |
| **"Just ask an agent"** (baseline) | Claude/Codex explores the repo and explains it | LLM exploration each time | Chat transcript | **no** | very high (repeated per person/session) | none (ephemeral) |

Adjacent non-competitors: API-reference generators (TypeDoc etc. — reference, not
orientation), static-site doc tools (Docusaurus — authoring, not analysis), GitHub code
search/graphs (navigation, not narrative).

## 2. Gap onboard fills

No tool in the table produces an **automatically generated, human-facing, interactive
tour** that is also a **machine-readable context artifact**:

- CodeTour has the right *consumption* model but tours are hand-written, go stale
  silently, and live only in VS Code.
- repomix / Aider maps have the right *generation* model (deterministic, token-aware) but
  produce agent food, not a human onboarding experience.
- The agent baseline produces good prose but is non-reproducible, costs full-exploration
  tokens per run, and leaves no artifact.

onboard = deterministic analysis (repomix/Aider side) → structured tour bundle → an
interactive static viewer (CodeTour side) + optional metered LLM narration/chat
(bounded version of the agent baseline). Differentiation claims and how they are made
measurable: DESIGN §1.2.

## 3. What onboard deliberately borrows

| Borrowed idea | Source | Where it lands |
|---|---|---|
| Step = anchor (file+line) + narration | CodeTour | DESIGN §5.3 `TourStep.anchor`; v2 CodeTour export (§15.2) keeps the model compatible |
| Rank code objects to fit a budget | Aider repo map | module ranking §7.2, callee significance scoring §6.10, BM25 retrieval §10.4 |
| Deterministic packing, no-LLM default | repomix | ADR-002, determinism contract §5.6 |
| Drift-awareness as a first-class problem | Swimm, CodeTour's weakness | content-hash excerpt ids now (§5.1) so v2 `onboard check` needs no schema break (§15.2 #1) |

## 4. Registry / naming check (evidence)

- `npm view onboard` (2026-07-10): **taken** — `onboard@0.0.10` exists. Decision: bin
  name stays `onboard`, published package name decided at release (ADR-001, KU-1).
  Candidates to check at that time: `onboard-tour`, `@<org>/onboard`.
- Current majors verified via `npm view` on 2026-07-10: typescript 7.0.2, ts-morph 28.0.0,
  shiki 4.3.1, preact 10.29.7, vite 8.1.4, commander 15.0.0, zod 4.4.3, sirv 3.0.2,
  vitest 4.1.10, markdown-it 14.3.0, ignore 7.0.5.

## 5. Design implications (summary)

1. Ship the interactive experience without an editor dependency → static web viewer (ADR-004).
2. Make determinism a *tested* property, not a slogan → golden/double-run CI gates (§14.4).
3. Make token efficiency *visible* → `tokenReport` vs naive-dump baseline (§8.5), chat
   per-answer usage (§10.7).
4. Keep the bundle agent-consumable → raw-text excerpts + structured index in
   `tour-bundle.json`, viewer-only HTML kept out of the canonical artifact (§9.1).
