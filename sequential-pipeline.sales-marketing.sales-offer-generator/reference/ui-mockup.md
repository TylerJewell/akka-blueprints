# UI mockup — sales-offer-generator

Single self-contained `src/main/resources/static-resources/index.html` (Lesson 17 — no `ui/`, no npm). Browser title: `<title>Akka Sample: Sales Offer Generator</title>`. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchored to `specs/vision/governance.html`). Content wraps in a `max-width:1080px` column with no horizontal scroll (Lesson 12).

Five tabs in the left nav: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Tab switching MUST be attribute-based (Lesson 26)

Switch tabs by matching `data-tab` (nav) to `data-panel` (section), never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Any tab removed during authoring must be deleted from the DOM, not hidden with `display:none` — a hidden zombie panel breaks index-based switching and shows a blank panel.

## Tab 1 — Overview
Eyebrow "Overview" + headline naming the sample type (sequential-pipeline sales offer generator), no subtitle. Four cards: **Try it** (the `/akka:build` run line), **How it works** (analyze → match → price → compose → review), **Components** (the inventory from `SPEC.md` Section 4), **API contract** (the top-level surface).

## Tab 2 — Architecture
Renders the four mermaid diagrams from `PLAN.md`. Include the Lesson 24 CSS: state-label colour `#ffffff !important` on every label DOM path, and `overflow:visible !important` on edge-label `foreignObject`; set `nodeTextColor`, `stateLabelColor`, and `transitionLabelColor` (`#cccccc`) in `mermaid.initialize`.

## Tab 3 — Risk Survey
Fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` style (Lesson 18): each answer is a row with the question as a yellow label and the value on the right. Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix
Fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control in the same style. The label column carries the control id plus a colored mechanism pill: `guardrail` red, `sanitizer` green. The value column shows name, rationale, and implementation.

## Tab 5 — App UI
- **Brief form**: a textarea for the customer brief and a Submit button. Submit POSTs `/api/offers` and clears the field.
- **Live offer list**: subscribes to `/api/offers/sse`. Each row shows the offer id (short), status pill, and quoted total when present.
- **Offer detail**: clicking a row expands to show the structured needs, matched products (sku · name · list price · fit reason), the pricing breakdown (subtotal · discount · total), and the rendered offer document. Rejected offers show the policy reason.
