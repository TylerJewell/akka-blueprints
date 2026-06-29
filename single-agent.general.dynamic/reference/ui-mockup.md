# UI mockup — Dynamic Route Agent

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown, YAML, and mermaid are acceptable. Browser title: `<title>Akka Sample: Dynamic Route Agent</title>`. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the governance.html anchor). Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

Five tabs in the left nav: Overview, Architecture, Risk Survey, Eval Matrix, App UI.

## Tab switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` / `data-panel` attribute, never by NodeList index. Each nav tab carries `data-tab="N"` and each panel carries `data-panel="N"`; the handler toggles `.active` by matching the attribute value:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM, never hidden with `display:none` — a zombie panel left in the DOM shifts the attribute mapping and blanks the App UI tab. There are exactly five panels, numbered 0-4.

## Tab 0 — Overview

Eyebrow "Overview" + headline naming the sample type (single-agent, configured per request) with no subtitle, then four cards:

- **Try it** — the single command `/akka:build`. No env-var export block.
- **How it works** — one generic agent, configured per request, routes SUMMARIZE and TRANSLATE; two guardrails bracket the call.
- **Components** — DynamicAgent, RequestEntity, RequestsView, RouteEndpoint, AppEndpoint, RequestSimulator.
- **API contract** — the top-level endpoint list.

## Tab 1 — Architecture

Renders the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, entity model). Carries the Lesson 24 state-label CSS overrides (white `color`/`fill` on every state-label DOM path, `overflow:visible` on edge-label foreignObjects) and the `transitionLabelColor: #cccccc` theme variable.

## Tab 2 — Risk Survey

Fetches `/api/metadata/risk-survey` and renders it in the `matrix-card` / `matrix-row` style: each answer is a row with the question/key on the left (yellow label) and the value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 3 — Eval Matrix

Fetches `/api/metadata/eval-matrix` and renders one `matrix-row` per control in the same style. The label column carries the control id plus a colored mechanism pill (`guardrail` red). The value column shows name, rationale, and implementation. Two controls: G1, G2.

## Tab 4 — App UI

The live surface:

- **Route selector** — a control choosing SUMMARIZE or TRANSLATE.
- **Text area** — the input text.
- **Target-language field** — shown only when route is TRANSLATE.
- **Submit button** — POSTs `SubmitRequest` to `/api/requests`.
- **Request list** — cards driven by `/api/requests/sse`, keyed by `id`, colored by status (SUBMITTED neutral, COMPLETED green, BLOCKED red). Each card shows route, input excerpt, status, and the `output` or `blockedReason` once present.
