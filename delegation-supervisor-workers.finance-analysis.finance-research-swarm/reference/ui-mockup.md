# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark, yellow accent, Instrument Sans, dot-grid background. Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

Browser title: `<title>Akka Sample: Finance Assistant Swarm Agent</title>`.

Five left-nav tabs. Each `.nav-tab` carries `data-tab="N"`; each `.tab-panel` carries `data-panel="N"`.

## Tab 1 — Overview

Eyebrow "Overview" + headline "Finance Assistant Swarm Agent", **no subtitle**. Four cards:

- **Try it** — the single command `/akka:build`. No env-var export block.
- **How it works** — four lines: resolve ticker, parallel fan-out to PriceAnalyst + NewsAnalyst + FundamentalsAnalyst, integrate with sector sanitizer and output guardrail.
- **Components** — the count: 5 autonomous-agent, 1 workflow, 2 event-sourced-entity, 1 view, 1 consumer, 2 timed-action, 2 http-endpoint.
- **API contract** — the top-level endpoints from `api-contract.md`.

## Tab 2 — Architecture

The four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER) plus a click-to-expand component table with syntax-highlighted Java snippets. Carry the Lesson 24 CSS overrides: white `fill`/`color` on every state-label DOM path, `overflow:visible` on edge-label `foreignObject`, and `transitionLabelColor:#cccccc` in `mermaid.initialize`.

## Tab 3 — Risk Survey

Renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style from `governance.html`. Question on the left (yellow label), answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Renders `/api/metadata/eval-matrix` in `matrix-card` / `matrix-row` style. The label column carries the control id and a colored mechanism pill (`guardrail` red, `sanitizer` orange, `eval-event` blue). Click a row to expand name, rationale, and implementation. Three controls: G1, S1, E1.

## Tab 5 — App UI

A form with one `query` text input (placeholder "Company name or ticker, e.g. Apple or AAPL") and a Submit button (`POST /api/reports`). Below it, a live list of reports streamed from `/api/reports/sse`. Each row shows the query, the resolved ticker symbol (once available), a status pill (RESOLVING / IN_PROGRESS / PUBLISHED / PARTIAL / BLOCKED), and the eval score when present. Click a row to expand: price summary table (5 price points), news headlines list, fundamentals key-value block, integrated report summary, and eval rationale.

## Tab switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` / `data-panel` attribute, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Exactly five `.tab-panel` sections exist in the DOM. Removed tabs are deleted, never hidden with `display:none` — a hidden zombie panel takes an index slot and blanks the App UI tab.
