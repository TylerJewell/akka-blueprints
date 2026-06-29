# UI mockup — Game Builder Team

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Browser title: `<title>Akka Sample: Game Builder Team</title>`. Visual style: dark background, yellow accent, Instrument Sans, dot-grid — matching `specs/vision/governance.html`. Content wraps in `.wrap{ max-width:1080px }` with no horizontal scroll.

Five tabs, in order.

## Tab 1 — Overview

Eyebrow "Overview" + headline naming the sample type (sequential pipeline, dev-code), no subtitle. Four cards:

- **Try it** — the single command `/akka:build` (no env-var export block).
- **How it works** — the engineer → QA → chief pipeline in three lines.
- **Components** — the agent, workflow, entity, view, consumer, timer, and endpoint inventory.
- **API contract** — the top-level endpoint list, pointing at the App UI tab.

## Tab 2 — Architecture

Renders the four mermaid diagrams from `reference/architecture.md` (component graph, sequence, state machine, ER). The `<style>` block carries the Lesson 24 overrides so state-diagram labels are white-on-dark and edge labels are not clipped: explicit `color`/`fill` on `.nodeLabel`, `.stateLabel`, `g.statediagram-state .label` paths, and `overflow:visible` on every edge-label `foreignObject`. `mermaid.initialize` sets `nodeTextColor`, `stateLabelColor`, and `transitionLabelColor:#cccccc`.

## Tab 3 — Risk Survey

Fetches `/api/metadata/risk-survey` and renders it in the `matrix-card` / `matrix-row` style. Each answer is a row: the question on the left (yellow label), the value on the right. Any value matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` renders in muted italic ("To be completed by deployer") so deployer-specific fields are visually distinct.

## Tab 4 — Eval Matrix

Fetches `/api/metadata/eval-matrix` and renders it in the same `matrix-card` / `matrix-row` style — one row per control. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `eval-event` blue, `ci-gate` pale yellow). The value column shows the control name, rationale, and implementation note.

## Tab 5 — App UI

- A brief input and a **Build** button that POSTs `/api/build-request`.
- A live list of builds via `EventSource('/api/builds/sse')`. Each row shows the brief, the status (with a colored chip), the filename, and the QA score when present.
- An expandable per-row panel showing the generated `sourceCode` and the QA `qaNotes` / `shipSummary` / `reworkReason`.
- No Approve/Reject buttons — the pipeline is autonomous; the human watches it run.

## Tab switching MUST be attribute-based (Lesson 26)

Tab switching matches by `data-tab` (nav) and `data-panel` (section) attributes, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

There are exactly five `.tab-panel` sections with `data-panel` values `0`–`4`. No removed tab is left hidden with `display:none` — dead panels are deleted from the DOM, never hidden, so no zombie panel can steal an index.
