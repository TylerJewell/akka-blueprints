# UI mockup

A single self-contained `src/main/resources/static-resources/index.html` (Lesson 17). Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Visual style anchors `specs/vision/governance.html`: dark background, yellow accent, Instrument Sans, dot-grid. Browser title `<title>Akka Sample: Content Creator Flow</title>`. Content fits the 1080px column with no horizontal scroll (Lesson 12).

## Tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample (not the flow), no subtitle. Four cards: Try it (`/akka:build`, then open the URL), How it works (one-topic-in, three-outputs-out), Components (the inventory from `SPEC.md` Section 4), API contract (the table from `api-contract.md`).
2. **Architecture** — the four mermaid diagrams from `PLAN.md`. Initialize mermaid with the Akka theme variables plus `nodeTextColor`, `stateLabelColor`, and `transitionLabelColor: #cccccc`, and include the CSS overrides forcing `color/fill #ffffff` on every state-label DOM path and `overflow: visible` on edge-label `foreignObject` elements (Lesson 24).
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Each answer is a row: question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic as "To be completed by deployer".
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `eval-event` blue). The value column shows name, rationale, and implementation. Two controls: G1, E1.
5. **App UI** — the live tab. A topic input and Submit button POST `/api/campaigns`. Below, a list of campaigns streamed over `/api/campaigns/sse`, newest first. Each campaign card shows: id (short), status badge, topic, and — when present — the research report, blog post, LinkedIn post, brand verdict, and quality score. `BLOCKED` campaigns show the block reason and withhold the outputs.

## Tab-switching MUST be attribute-based (Lesson 26)

Switch tabs by matching `data-tab` on the nav element to `data-panel` on the panel — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Any tab removed during authoring must be deleted from the DOM, not hidden with `display:none`. A hidden panel still appears in `querySelectorAll('.tab-panel')` and, under index-based switching, would shift the App UI panel to the wrong index and leave it blank.
