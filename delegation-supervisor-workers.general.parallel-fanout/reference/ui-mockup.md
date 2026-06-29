# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark, yellow accent, Instrument Sans, dot-grid background. Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

Browser title: `<title>Akka Sample: Parallel Task Decomposition</title>`.

Five left-nav tabs. Each `.nav-tab` carries `data-tab="N"`; each `.tab-panel` carries `data-panel="N"`.

## Tab 1 — Overview

Eyebrow "Overview" + headline "Parallel Task Decomposition", **no subtitle**. Four cards:

- **Try it** — the single command `/akka:build`. No env-var export block.
- **How it works** — three lines: decompose the job payload, parallel fan-out to SubtaskWorkerA + SubtaskWorkerB, consolidate with output validation.
- **Components** — the count: 3 autonomous-agent, 1 workflow, 2 event-sourced-entity, 1 view, 1 consumer, 2 timed-action, 2 http-endpoint.
- **API contract** — the top-level endpoints from `api-contract.md`.

## Tab 2 — Architecture

The four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER) plus a click-to-expand component table with syntax-highlighted Java snippets. Carry the Lesson 24 CSS overrides: white `fill`/`color` on every state-label DOM path, `overflow:visible` on edge-label `foreignObject`, and `transitionLabelColor:#cccccc` in `mermaid.initialize`.

## Tab 3 — Risk Survey

Renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style from `governance.html`. Question on the left (yellow label), answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Renders `/api/metadata/eval-matrix` in `matrix-card` / `matrix-row` style. The label column carries the control id and a colored mechanism pill (`guardrail` red, `eval-event` blue). No controls are defined for this baseline; the tab renders an empty state message: "No governance controls are defined for this baseline."

## Tab 5 — App UI

A form with one `payload` text input and a Submit button (`POST /api/tasks`). Below it, a live list of jobs streamed from `/api/tasks/sse`. Each row shows the payload (truncated to 80 chars), a status pill (QUEUED / PROCESSING / CONSOLIDATED / DEGRADED / REJECTED), and the quality score when present. Click a row to expand the structural elements, contextual enrichment, the consolidated summary, and the quality rationale.

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
