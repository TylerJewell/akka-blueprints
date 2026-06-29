# UI mockup — screenplay-writer

A single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Inline CSS and JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Screenplay Writer</title>`. Visual style follows `specs/vision/governance.html` — dark background, yellow accent, Instrument Sans, dot-grid.

Five left-nav tabs. Each `.nav-tab` carries `data-tab="0..4"`; each `.tab-panel` carries the matching `data-panel`.

## Tab 1 — Overview (`data-panel="0"`)

Eyebrow "Overview" + headline naming the sample type (sequential pipeline), no subtitle. Four cards: **Try it** (just `/akka:build`), **How it works** (the four pipeline stages), **Components** (the inventory), **API contract** (the top-level endpoints).

## Tab 2 — Architecture (`data-panel="1"`)

The four mermaid diagrams from `PLAN.md`, each in a `.diagram-card`. Apply the Lesson 24 CSS overrides: force `color`/`fill` white on every state-label DOM path, set `overflow:visible` on edge-label `foreignObject`, and pass `nodeTextColor` / `stateLabelColor` / `transitionLabelColor: #cccccc` to `mermaid.initialize`.

## Tab 3 — Risk Survey (`data-panel="2"`)

Fetches `/api/metadata/risk-survey` and renders it as `matrix-card` / `matrix-row` pairs — question label (yellow) on the left, answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic with placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix (`data-panel="3"`)

Fetches `/api/metadata/eval-matrix` and renders one `matrix-row` per control. The label column carries the control id plus a coloured mechanism pill: `sanitizer` green, `guardrail` red. The value column shows name, rationale, and implementation.

## Tab 5 — App UI (`data-panel="4"`)

- A **thread** textarea and an optional **source label** input.
- A **Generate screenplay** button that POSTs to `/api/threads`.
- A live **job list** fed by `/api/jobs/sse`, each row showing source label, status pill, and redaction count.
- A **detail panel** for the selected job: the character list, the synopsis, the formatted screenplay in a monospace block, and — when status is `BLOCKED` — the `piiFindings` shown as a red notice.

## Tab switching MUST be attribute-based

Per Lesson 26, switch tabs by matching `data-tab` to `data-panel`, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM, never hidden with `display:none`. A zombie panel left in the DOM shifts NodeList indices and blanks the App UI tab.
