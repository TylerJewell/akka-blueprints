# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Browser title `<title>Akka Sample: Prospect Analysis</title>`. Visual style anchor: dark background, yellow accent, Instrument Sans, dot-grid (the `governance.html` style).

Left nav with five tabs.

## Tab 1 — Overview
Eyebrow "Overview" + headline naming the sample type, no subtitle. Four cards: Try it (`/akka:build`, no env-var export block), How it works, Components, API contract.

## Tab 2 — Architecture
Renders the four PLAN.md mermaid diagrams (component graph, sequence, state machine, ER) with a paragraph of context each. Includes the Lesson 24 CSS overrides — state-label colour forced white, edge-label `foreignObject` `overflow:visible`, `transitionLabelColor` `#cccccc`.

## Tab 3 — Risk Survey
Renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style: question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic.

## Tab 4 — Eval Matrix
Renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `sanitizer` green). The value column shows name, rationale, and implementation.

## Tab 5 — App UI
- A company-name input, an optional domain input, and an Analyze button. Submitting POSTs `/api/analyze`.
- A live list of prospects fed by `/api/prospects/sse`, each row showing company name and a status chip (`RESEARCHING` / `ANALYZING` / `STRATEGIZING` / `COMPLETED` / `STALLED`).
- Each row expands to show the company profile, the decision-maker list with masked contact hints, and the outreach strategy once each phase completes.

## Tab-switching MUST be attribute-based (Lesson 26)

Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM. No `display:none` zombie panels — a hidden panel still occupies a NodeList position and breaks index-based switching.
