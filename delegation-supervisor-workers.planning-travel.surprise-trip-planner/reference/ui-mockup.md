# UI mockup — surprise-trip-planner

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for `marked` and `js-yaml` are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the `governance.html` anchor). Browser title: `<title>Akka Sample: Surprise Trip Planner</title>`.

Five left-nav tabs.

## Tab 1 — Overview

Eyebrow "Overview" + headline naming the sample type (delegation supervisor-workers travel planner), no subtitle. Four cards:

- **Try it** — one line: run `/akka:build`, open the listed URL, submit a preference set.
- **How it works** — supervisor delegates to three workers, then assembles.
- **Components** — the agent / workflow / entity / view / consumer / timed-action / endpoint inventory.
- **API contract** — the top-level endpoint list.

## Tab 2 — Architecture

Renders the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER). Include the Lesson 24 CSS overrides: white `color`/`fill` on every state-label DOM path, `overflow: visible` on every edge-label `foreignObject`, and `transitionLabelColor: #cccccc` plus `nodeTextColor` / `stateLabelColor` in `mermaid.initialize`.

## Tab 3 — Risk Survey

Fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` style — question as the yellow left label, answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic.

## Tab 4 — Eval Matrix

Fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`guardrail` red). The value column shows name, rationale, and implementation.

## Tab 5 — App UI

- **Preference form:** `vibe` (text), `budgetBand` (select low/mid/high), `startDate` + `endDate` (date), `partySize` (number), `noGo` (comma-separated text). A **Plan my surprise** button POSTs `/api/trips`.
- **Live trip list:** subscribes to `/api/trips/sse`, one card per trip keyed by `id`, showing status. While status is `REQUESTED`/`RESEARCHED`, the destination is masked ("planning your surprise…"). On `READY`, the card reveals destination, logistics, and the day-by-day activities; `groundingNotes` and any `blockedToolNote` show as small footnotes. `ESCALATED` and `FAILED` show their reason.

## Tab-switching MUST be attribute-based (Lesson 26)

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

Removed tabs are deleted from the DOM, never hidden with `display:none`. No zombie panels — an index-based switcher plus a stale hidden panel leaves the App UI tab blank.
