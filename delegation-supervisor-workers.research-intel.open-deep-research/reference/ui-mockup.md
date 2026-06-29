# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark, yellow accent, Instrument Sans, dot-grid background. Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

Browser title: `<title>Akka Sample: Open Deep Research</title>`.

Five left-nav tabs. Each `.nav-tab` carries `data-tab="N"`; each `.tab-panel` carries `data-panel="N"`.

## Tab 1 — Overview

Eyebrow "Overview" + headline "Open Deep Research", **no subtitle**. Four cards:

- **Try it** — the single command `/akka:build`. No env-var export block.
- **How it works** — three lines: manager decomposes the question into a retrieval plan; WebBrowsingAgent fetches URLs (guardrail checks each before execution) and FileReaderAgent reads the document in parallel; manager synthesises a cited answer from sanitized content.
- **Components** — the count: 3 autonomous-agent, 1 workflow, 2 event-sourced-entity, 1 view, 1 consumer, 2 timed-action, 2 http-endpoint.
- **API contract** — the top-level endpoints from `api-contract.md`.

## Tab 2 — Architecture

The four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER) plus a click-to-expand component table with syntax-highlighted Java snippets. Carry the Lesson 24 CSS overrides: white `fill`/`color` on every state-label DOM path, `overflow:visible` on edge-label `foreignObject`, and `transitionLabelColor:#cccccc` in `mermaid.initialize`.

## Tab 3 — Risk Survey

Renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style from `governance.html`. Question on the left (yellow label), answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Renders `/api/metadata/eval-matrix` in `matrix-card` / `matrix-row` style. The label column carries the control id and a colored mechanism pill (`guardrail` red, `sanitizer` orange, `eval-event` blue, `hotl` purple). Click a row to expand name, rationale, and implementation. Four controls: G1, S1, E1, H1.

## Tab 5 — App UI

A form with one `question` text input and a Submit button (`POST /api/research`). Below it, a live list of runs streamed from `/api/research/sse`. Each row shows the question (truncated at 80 characters), a status pill (QUEUED / IN_PROGRESS / ANSWERED / DEGRADED / BLOCKED), the oversight flag icon when `oversightFlagged` is true, and the eval score when present. Click a row to expand: web pages (URL + title + extracted text), document sections (heading + content), the synthesised answer with citations, and the eval rationale. The blocked URL, if any, is shown with a red "blocked" label.

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
