# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchor: `specs/vision/governance.html`). Browser title: `<title>Akka Sample: HITL Story Crafting</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — start a story with a premise; live list of stories via SSE; per-story chapter list rendered in order; Continue and End buttons shown only when `status == AWAITING_DIRECTION`. Continue reveals a direction text field. Completed stories show all chapters. Screened-out stories show the guard reason.

## Tab switching MUST be attribute-based (Lesson 26)

Match nav tabs to panels by `data-tab` / `data-panel` attribute, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM, not hidden with `display:none`. No zombie panels — a hidden panel still occupies a `querySelectorAll` index and breaks index-based switching.

## App UI fields and controls

- **Premise input** — single text field + Start Story button → `POST /api/stories`.
- **Stories list** — one card per story: premise excerpt, status badge, turn count, and (for `AWAITING_DIRECTION` stories) a direction text field plus Continue and End buttons.
- **Chapter list** — inside each story card, chapters are listed in `turnNumber` order with their title and body. Direction text from the reader is shown above the chapter it influenced (turn N+1 direction shown between chapter N and chapter N+1).
- **Status badge** — `AWAITING_DIRECTION` neutral, `DRAFTING` yellow (blinking), `COMPLETED` green, `SCREENED_OUT` muted red.
- **Continue flow** — clicking Continue reveals the direction field; submitting POSTs `{ direction }` to `/api/stories/{id}/continue`.
- **End flow** — clicking End POSTs to `/api/stories/{id}/end` with no body.
