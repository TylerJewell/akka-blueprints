# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Trip Planner Team</title>`. Visual anchor: dark background, yellow accent, Instrument Sans, dot-grid — matching the governance style. Content wraps in `.wrap{ max-width:1080px }` with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow ("Overview") + headline naming the sample type (not the flow), no subtitle, then four cards: Try it (`/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — the four `PLAN.md` mermaid diagrams (component graph, sequence, state machine, ER). Apply the Lesson 24 CSS overrides so state-diagram labels render white and edge labels are not clipped (`overflow:visible` on edge-label `foreignObject`, `transitionLabelColor` `#cccccc`).
3. **Risk Survey** — renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style; the question is the yellow left label, the answer the right value. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style, one row per control. The label column carries the control id plus a colored mechanism pill (`guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — the live surface (below).

## App UI tab

- **Submit form:** `destination` (text), `preferences` (textarea), `days` (number, default 3), Submit button → `POST /api/trips`.
- **Live trip list:** subscribes to `/api/trips/sse`; one card per trip showing destination, a status chip (`GATHERING_INSIGHTS` / `PLANNING_ITINERARY` / `PLANNING_LOGISTICS` / `COMPLETED` / `FAILED`), and four progressively-filled sections: Insights, Itinerary, Logistics, Final plan. Empty sections show a muted placeholder until their event fires.
- On `FAILED`, the card shows `failureReason`.

## Tab switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` / `data-panel` attribute, never by NodeList index. Removed tabs are deleted from the DOM — never hidden with `display:none` (hidden panels become zombies that steal index positions and blank out the App UI tab).

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```
