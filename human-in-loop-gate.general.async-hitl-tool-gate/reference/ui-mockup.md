# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Browser title: `<title>Akka Sample: Async HITL Tool Gate</title>`. Visual style anchor: `specs/vision/governance.html` (dark / yellow accent / Instrument Sans / dot-grid).

## Five tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample type (not the flow), no subtitle. Four cards: Try it (the single command `/akka:build`), How it works, Components, API contract.
2. **Architecture** — the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER). Mermaid initialized with the Akka theme variables and the Lesson 24 CSS overrides so state-box labels are white-on-dark and edge labels are not clipped.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Question on the left (yellow label), answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic ("To be completed by deployer").
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style, one row per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — submit a request; live SSE list of actions; per-action Approve/Reject buttons shown only when status is `PLANNED`. Each action card shows `toolName`, `toolArguments`, `rationale`, and — once executed — `toolOutput`.

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

Removed tabs must be **deleted** from the DOM, never hidden with `display:none` — a hidden zombie panel still occupies a `querySelectorAll` index and breaks index-based switching. There are exactly five `.tab-panel` sections, each with a `data-panel` matching its nav tab's `data-tab`.
