# UI mockup — content-pipeline

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build (Lesson 17). Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Browser title: `<title>Akka Sample: Content Pipeline</title>`. Visual style follows `specs/vision/governance.html` (dark, yellow accent, Instrument Sans, dot-grid background).

Five tabs in the left nav: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Tab 1 — Overview

Eyebrow "Overview" + headline naming the sample type (sequential content pipeline), **no subtitle**. Then four cards:

- **Try it** — the `/akka:build` command and a one-line "open http://localhost:9942 and submit a topic". No env-var export block.
- **How it works** — research → write → critique → publish in one durable workflow.
- **Components** — the component inventory (links to the Architecture tab).
- **API contract** — the top-level endpoint list.

## Tab 2 — Architecture

Renders the four mermaid diagrams from `architecture.md` (component graph, sequence, state machine, ER). Mermaid initialized with the Akka theme variables AND the Lesson 24 CSS overrides: explicit white `color`/`fill` on every state-label DOM path, and `overflow: visible` on every edge-label `foreignObject`. A click-to-expand component table maps each component to its Akka primitive.

## Tab 3 — Risk Survey

Renders `risk-survey.yaml` in the `matrix-card` / `matrix-row` style from `governance.html`. Each answer is a row: the question label on the left (yellow), the value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Renders `eval-matrix.yaml` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `eval-event` blue). The value column shows name, rationale, and implementation. Three controls: G1, G2, E1.

## Tab 5 — App UI

- A topic input box and a **Submit** button (`POST /api/topics`).
- A live list of articles via the `/api/articles/sse` stream.
- Each article shows a stage timeline: RESEARCHING → WRITING → CRITIQUING → PUBLISHED (or FAILED), with the current status highlighted.
- Per article, expandable detail: research summary + sources, draft title + body, critique score + notes, and the published URL once present.
- A FAILED article shows its failure reason.

## Tab-switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` → `data-panel` attribute, never by NodeList index. Removed tabs must be deleted from the DOM, never hidden with `display:none` (no zombie panels).

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Each `.nav-tab` carries `data-tab="0".."4"` and each `.tab-panel` carries the matching `data-panel`. The nav-tab → panel mapping stays correct even if a stray panel reappears.
