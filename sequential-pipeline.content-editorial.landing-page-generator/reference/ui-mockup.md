# UI mockup — Landing Page Generator

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm, no build step. Inline CSS + JS; runtime CDN imports for markdown, YAML, and mermaid are allowed. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the `governance.html` anchor). Content column max-width 1080px, no horizontal scroll (Lesson 12).

Browser title: `<title>Akka Sample: Landing Page Generator</title>`.

## Tab-switching MUST be attribute-based (Lesson 26)

Switch tabs by matching `data-tab` on the nav item to `data-panel` on the section — never by NodeList index. Removed tabs are deleted from the DOM, never `display:none`-ed into zombies.

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, no subtitle), then four cards: **Try it** (run `/akka:build`, then open `http://localhost:9297/`), **How it works** (the copy → structure → cta → review pipeline in one line), **Components** (the eleven components), **API contract** (the endpoint list from `api-contract.md`).
2. **Architecture** — the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER). Apply the Lesson 24 state-label CSS overrides and theme variables (`nodeTextColor`/`stateLabelColor` white, `transitionLabelColor` `#cccccc`, edge-label `foreignObject { overflow:visible }`).
3. **Risk Survey** — fetch `/api/metadata/risk-survey`, render in `matrix-card` / `matrix-row` style: question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic as "To be completed by deployer".
4. **Eval Matrix** — fetch `/api/metadata/eval-matrix`, render in the same style, one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill — **guardrail is red**. The value column shows name, rationale, and implementation.
5. **App UI** — the live surface:
   - **Concept input** — a text field "Describe your product or service" + a **Generate page** button. Posts to `/api/concepts`.
   - **Pages list** — driven by `/api/pages/sse`. Each row shows the concept, a status pill (`DRAFTING_COPY` / `STRUCTURING` / `WRITING_CTA` / `REVIEWING` / `READY` / `FLAGGED`), and the review score when present.
   - **Ready page detail** — for `READY` pages, render the headline, subhead, the ordered section list, and the primary/secondary CTA.
   - **Flagged page detail** — for `FLAGGED` pages, show the `flagReason` in red.
