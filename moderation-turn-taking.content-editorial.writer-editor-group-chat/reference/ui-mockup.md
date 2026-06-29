# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the `governance.html` anchor). Browser title `<title>Akka Sample: Writer-Editor Group Chat</title>`. Content wraps in a 1080px column with no horizontal scroll.

Five tabs in the left nav: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Tab switching MUST be attribute-based (Lesson 26)

Each nav item carries `data-tab="<key>"`; each panel carries `data-panel="<key>"` with matching keys (`overview`, `architecture`, `risk`, `eval`, `app`). Switching toggles `.active` by matching `data-tab` to `data-panel` — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

There are exactly five `.tab-panel` sections. Removed tabs are deleted from the DOM, never hidden with `display:none` — no zombie panels.

## Tabs

### Overview
Eyebrow "Overview" + headline naming the sample type (not the flow), no subtitle. Four cards: **Try it** (just `/akka:build` — no env-var export block), **How it works** (the round-robin manager in two sentences), **Components** (the component list), **API contract** (the `/api` surface).

### Architecture
The four mermaid diagrams from `PLAN.md` — component graph, interaction sequence, state machine, entity model. Mermaid is initialized with the Lesson 24 theme variables (`nodeTextColor`, `stateLabelColor`, `transitionLabelColor: #cccccc`) and the CSS overrides forcing `color/fill:#ffffff` on every state-label DOM path and `overflow:visible` on every edge-label `foreignObject`. Below the diagrams, a click-to-expand component table.

### Risk Survey
Renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Each answer is a row: question on the left (yellow label), value on the right. Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer").

### Eval Matrix
Renders `/api/metadata/eval-matrix` in the `matrix-card` / `matrix-row` style, one row per control. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `eval-event` blue, `ci-gate` pale yellow, `halt` red). The value column shows name, rationale, and implementation. Click a row to expand the full rationale.

### App UI
The live sample.

- **Topic form** — a text input and a **Submit** button. Submit POSTs `/api/conversations`; the new conversation appears in the transcript list.
- **Transcript list** — conversations streamed from `/api/conversations/sse`, newest first. Each conversation shows its topic, a status banner (`IN_PROGRESS` / `APPROVED` / `MAX_ROUNDS_REACHED` / `ESCALATED`), and its ordered turns.
- **Turn rows** — each turn is tagged `WRITER` or `EDITOR` with its round number. Editor turns show the eval score. A blocked turn is shown muted with its blocked reason. On `APPROVED`, the final draft is highlighted.
