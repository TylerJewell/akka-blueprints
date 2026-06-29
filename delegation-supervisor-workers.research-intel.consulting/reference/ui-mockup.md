# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build, no separate UI build step. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Browser title: `<title>Akka Sample: Consulting Coordinator</title>`. Visual style anchor: `specs/vision/governance.html` (dark / yellow accent / Instrument Sans / dot-grid background), fitting the 1080px content column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample type (no subtitle), then four cards: **Try it** (`/akka:build`, then open the App UI tab — no env-var export block), **How it works** (route → delegate-or-handoff → deliver, with compliance review and routing eval), **Components** (the inventory from `PLAN.md`), **API contract** (the top-level surface).
2. **Architecture** — the four mermaid diagrams from `PLAN.md` with the Lesson 24 mermaid CSS overrides (state-label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`) and a click-to-expand component table.
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` style. Each answer is a row: question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders in `matrix-card` / `matrix-row` style, one row per control. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `hotl` muted, `eval-event` blue). The value column shows name, rationale, and implementation.
5. **App UI** — submit an engagement brief; live SSE list of engagements. Each card shows route, complexity score, routing rationale, eval score, deliverable title/content, and — on `DELIVERED` senior recommendations (`assignedTo = senior`) — a compliance-review form (reviewer, PASS/FLAG, notes) that POSTs to `/api/engagements/{id}/compliance`.

## App UI fields and actions

- **Brief input** — a textarea + **Submit** button → `POST /api/engagements`.
- **Engagement list** — one card per engagement, upserted by `id` from the SSE stream.
- **Status pill** — colored by `status`.
- **Compliance form** — shown only when `status == DELIVERED` and `assignedTo == senior`; reviewer text field, PASS/FLAG selector, notes textarea, **Record review** button.

## Tab-switching MUST be attribute-based (Lesson 26)

Switch tabs by matching the `data-tab` attribute on nav items to the `data-panel` attribute on panels — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Any tab removed during iteration must be **deleted** from the DOM, not hidden with `display:none`. A hidden panel still appears in `querySelectorAll('.tab-panel')` and, under index-based switching, shifts the real panels so a nav click activates a zombie panel and the user sees a blank screen. Attribute-based switching plus DOM deletion prevents both halves of that failure.
