# UI mockup — rag-knowledge-agent

One self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the governance.html anchor). Content wraps in a 1080px column with no horizontal scroll. Browser title: `<title>Akka Sample: RAG Knowledge Agent</title>`.

Five left-nav tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Tab 1 — Overview

Eyebrow "Overview" + headline naming the sample type (single-agent grounded question answering), no subtitle. Four cards:
- **Try it** — open the App UI tab, ask a question, read the cited answer; the Try-it card names `/akka:build` only, with no env-var export block.
- **How it works** — retrieve passages, answer from them, refuse when unsupported, score faithfulness.
- **Components** — the nine components from SPEC Section 4.
- **API contract** — the top-level surface from `api-contract.md`.

## Tab 2 — Architecture

Renders the four mermaid diagrams from `PLAN.md`. Apply the Lesson 24 CSS overrides (state-label color + fill `#ffffff`, edge-label `foreignObject { overflow:visible }`) and theme variables (`nodeTextColor`, `stateLabelColor`, `transitionLabelColor:#cccccc`) so state names and transition labels stay legible.

## Tab 3 — Risk Survey

Renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Each answer is a row: question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill: `guardrail` red, `eval-event` blue. The value column shows name, rationale, and implementation.

## Tab 5 — App UI

- A question box + **Ask** button. Submitting POSTs `/api/ask`.
- A live session list streamed from `/api/sessions/sse`, newest first, each row keyed by `id`.
- Each row shows the question and a status pill (`RECEIVED` / `RETRIEVED` / `ANSWERED` / `REFUSED` / `EVALUATED`).
- `ANSWERED` / `EVALUATED` rows show the answer, its citations (docTitle + snippet), and the faithfulness score when present.
- `REFUSED` rows show the refusal reason in muted text.

## Tab switching — MUST be attribute-based (Lesson 26)

Switch tabs by matching `data-tab` on the nav item to `data-panel` on the section — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs must be deleted from the DOM, not hidden with `display:none`. No zombie `.tab-panel` sections may remain.
