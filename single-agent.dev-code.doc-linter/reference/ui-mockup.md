# UI mockup — doc-linter

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder,
no npm build, no separate UI build step. Inline CSS + JS; runtime CDN imports for markdown
(`marked`) and YAML (`js-yaml`) are acceptable. Browser title:
`<title>Akka Sample: Doc Linter</title>`. Visual style anchor: dark / yellow accent /
Instrument Sans / dot-grid background. Content wraps in `.wrap{ max-width:1080px }` with no
horizontal scroll.

## Tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample type (single-agent doc
   linter), no subtitle, then four cards: Try it (`/akka:build`, then open
   `http://localhost:9251/`), How it works (one agent + custom lint tool + documentation
   gate), Components (the inventory), API contract (the top-level surface).
2. **Architecture** — the four mermaid diagrams from `PLAN.md`. Apply the Lesson 24 CSS
   overrides: explicit white `color`/`fill` on every state-label DOM path, `overflow:visible`
   on every edge-label `foreignObject`, and `transitionLabelColor #cccccc` in
   `mermaid.initialize`.
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` /
   `matrix-row` style: question label (yellow) left, answer right. Values matching
   `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per
   control. The label column carries the control id plus a colored mechanism pill: guardrail
   red, ci-gate pale yellow. The value column shows name, rationale, and implementation.
5. **App UI** — the live surface (below).

## App UI tab

- **File picker** — a select populated from `GET /api/files`, plus a free-text field for a
  custom path (used to demonstrate the path-traversal block). A **Lint** button POSTs
  `/api/lint`.
- **Run list** — fed by `GET /api/lint/sse`, newest first. Each row shows the file path,
  the status, and a **gate badge** (green PASS / red FAIL / grey for non-LINTED). `BLOCKED`
  rows show the block reason in red.
- **Run detail** — clicking a run expands a findings table (rule, line, severity, message)
  in the agent's ranked order, with the summary above it.
- **Feedback** — each finding row has a small "too strict" / "keep flagging" control that
  POSTs `/api/lint/{id}/feedback`; the run moves to `FEEDBACK_APPLIED`.

## Tab switching — MUST be attribute-based (Lesson 26)

Switch tabs by matching `data-tab` on the nav element to `data-panel` on the section, never
by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs must be **deleted** from the DOM, not hidden with `display:none`. A hidden
`<section class="tab-panel">` still appears in `querySelectorAll` and breaks index-based
switching — do not leave zombie panels.
