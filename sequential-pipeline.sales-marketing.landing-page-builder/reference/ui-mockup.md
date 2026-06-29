# UI mockup — landing-page-builder

A single self-contained `src/main/resources/static-resources/index.html` (Lesson
17 — no `ui/`, no npm build). Inline CSS + JS; runtime CDN imports for markdown
and YAML are fine. Browser title: `<title>Akka Sample: Landing Page Generator</title>`.
Visual style: dark background, yellow accent, Instrument Sans, dot-grid.

## Tabs

Five tabs in the left nav: Overview / Architecture / Risk Survey / Eval Matrix /
App UI.

1. **Overview** — eyebrow "Overview" + headline naming the sample (sequential
   pipeline), no subtitle, then four cards: Try it (`/akka:specify @SPEC.md`),
   How it works, Components, API contract.
2. **Architecture** — the four mermaid diagrams from `PLAN.md`, each with a line
   of context. State-diagram label colour and edge-label `overflow:visible` CSS
   overrides per Lesson 24.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in `matrix-card` /
   `matrix-row` style. Question on the left (yellow label), answer on the right.
   Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render muted-italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style. The
   label column carries the control id plus a colored mechanism pill (`guardrail`
   red, `ci-gate` pale yellow). The value column shows name, rationale, and
   implementation.
5. **App UI** — the live interaction:
   - A concept input field and a Submit button → POST `/api/generate`.
   - A live list of pages over `/api/pages/sse`. Each row shows id (short),
     concept, status pill, and template name when present.
   - Clicking a page opens its detail: brief, chosen template + source URL, lint
     result, and a rendered preview of the page markup in a sandboxed frame.
   - Rejected pages show the reject reason.

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

Removed tabs are deleted from the DOM, never hidden with `display:none` — a
hidden zombie panel takes an index slot and blanks the wrong tab.
