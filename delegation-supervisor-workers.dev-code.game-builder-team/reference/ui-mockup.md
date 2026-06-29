# UI mockup — game-builder-team

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid. Browser title: `<title>Akka Sample: Game Builder Team</title>`. Content fits the 1080px column with no horizontal scroll.

## Tab 1 — Overview
Eyebrow ("Overview") + headline (sample type, no subtitle) + four cards:
- **Try it** — the `/akka:build` line; no env-var export block.
- **How it works** — the design → code → test → assemble path in two sentences.
- **Components** — the agent/workflow/entity inventory.
- **API contract** — the top-level endpoint list.

## Tab 2 — Architecture
Renders the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER). Each diagram in a `.diagram-card`. Apply the Lesson 24 CSS overrides: explicit `color`/`fill:#ffffff` on every state-label DOM path, `overflow:visible` on every edge-label `foreignObject`, and `transitionLabelColor:#cccccc` plus `nodeTextColor`/`stateLabelColor` in `mermaid.initialize`.

## Tab 3 — Risk Survey
Fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` style: question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix
Fetches `/api/metadata/eval-matrix`, renders in the same `matrix-card` / `matrix-row` style, one row per control. The label column carries the control id and a colored mechanism pill: `guardrail` red, `ci-gate` pale yellow. The value column shows name, rationale, and implementation.

## Tab 5 — App UI
- A form: a single text input for the game idea and a "Build" button → `POST /api/games`.
- A live list of projects via `GET /api/games/sse`, newest first. Each card shows status, idea, and — once present — the spec title and mechanics.
- For `DELIVERED` projects a **Play** button loads `code.html` into a sandboxed `<iframe srcdoc>` (the `sandbox` attribute set, without `allow-same-origin`).
- For `BLOCKED` projects the `blockReason` shows; for `TEST_FAILED` the `testReport.failures` list shows.

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

Removed tabs are deleted from the DOM, never hidden with `display:none`. A stale hidden panel takes an index slot and silently breaks index-based switching, leaving a blank panel with no error.
