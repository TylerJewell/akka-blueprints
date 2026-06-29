# UI mockup — Meeting Prep Briefer

One self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm, no build step. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid — matching `specs/vision/governance.html`.

Browser title: `<title>Akka Sample: Meeting Prep Briefer</title>`.

## Tabs

Five tabs in the left nav.

### Overview
Eyebrow "Overview" + headline naming the sample type (not the flow), no subtitle. Then four cards:
- **Try it** — `/akka:build`, then open `http://localhost:9382`. No env-var export block.
- **How it works** — a supervisor plans research; workers research participants and the topic; a composer assembles the briefing.
- **Components** — the component inventory.
- **API contract** — the top-level endpoint list.

### Architecture
The four mermaid diagrams (component graph, sequence, state machine, entity model) from `PLAN.md`, each in a `.diagram-card`. Apply the Lesson 24 CSS overrides: state-diagram labels white-on-dark via the full `g.statediagram-state .label` DOM-path selector list, edge-label `foreignObject { overflow: visible }`, and `transitionLabelColor: #cccccc` in `mermaid.initialize`.

### Risk Survey
Fetches `/api/metadata/risk-survey`, renders it in `matrix-card` / `matrix-row` style — question on the left as a yellow label, answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

### Eval Matrix
Fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill — `guardrail` red, `sanitizer` green. The value column shows name, rationale, and implementation. Rows expand on click to show the full rationale.

### App UI
The live surface:
- A form: a text input for **meeting topic**, a repeatable text input (or comma-separated field) for **participants**, and a **Prepare briefing** button that POSTs `/api/briefings`.
- A live list of briefings via `EventSource('/api/briefings/sse')`. Each row shows the topic, the participant count, and a status chip (RECEIVED / PLANNING / RESEARCHING / COMPOSING / READY / FAILED).
- Clicking a `READY` briefing expands it to render `briefingDoc` as markdown plus the talking-points and questions lists.

## Tab switching — MUST be attribute-based (Lesson 26)

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

Any removed tab must be **deleted** from the DOM, not hidden with `display:none`. A hidden zombie panel still appears in `querySelectorAll('.tab-panel')` and breaks index-based switching; deleting it keeps the `data-tab` → `data-panel` mapping correct.
