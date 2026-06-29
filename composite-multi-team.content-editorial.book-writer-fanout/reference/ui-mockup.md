# UI mockup — book-writer-fanout

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder,
no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are fine. Visual
style anchor: `specs/vision/governance.html` (dark, yellow accent, Instrument Sans, dot-grid).
Content wraps in `.wrap { max-width: 1080px }` with no horizontal scroll. Browser title:
`<title>Akka Sample: Book Writer</title>`.

Five left-nav tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Tab 1 — Overview

Eyebrow "Overview" + headline naming the sample type (composite multi-team authoring), no
subtitle. Four cards:
- **Try it** — the single `/akka:specify @SPEC.md` line, then `/akka:build`. No env-var export block.
- **How it works** — outline → per-chapter writing fan-out → completeness gate → consolidate.
- **Components** — the agent, workflow, entity, view, consumer, timed-action, endpoint inventory.
- **API contract** — the top-level endpoint list, linking the App UI tab.

## Tab 2 — Architecture

Renders the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine,
ER). Each diagram is wrapped in a `.diagram-card .mermaid`. `mermaid.initialize` sets the Akka
theme variables plus `nodeTextColor`, `stateLabelColor`, and `transitionLabelColor: #cccccc`;
the `<style>` block adds the Lesson-24 overrides forcing white fill on every state-label DOM
path and `overflow: visible` on every edge-label `foreignObject`.

## Tab 3 — Risk Survey

Fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` style: question
on the left (yellow label), answer on the right. Any value matching
`/TO_BE_COMPLETED_BY_DEPLOYER/i` renders in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column
carries the control id plus a colored mechanism pill — `guardrail` red, `eval-event` blue,
`ci-gate` pale yellow. The value column shows name, rationale, and implementation.

## Tab 5 — App UI

- **Submit** — a text input for a book topic and a Submit button → `POST /api/books`.
- **Book list** — live over `GET /api/books/sse`. Each row shows the topic, status, and a
  `chaptersDrafted / chapterCount` progress indicator.
- **Expand a book** — reveals the title, the numbered chapter list with each chapter's status
  and quality score, and, once `COMPLETED`, the consolidated manuscript rendered as markdown.

## Tab-switching MUST be attribute-based (Lesson 26)

Nav tabs carry `data-tab="overview|architecture|risk|eval|app"`; panels carry the matching
`data-panel`. Switching toggles `.active` by comparing `dataset.tab` to `dataset.panel` —
never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM, never `display:none`. No zombie panels — an index-based
switcher plus a hidden leftover panel is exactly the blank-App-UI failure Lesson 26 records.
