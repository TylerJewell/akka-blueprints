# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchor: `specs/vision/governance.html`). Browser title: `<title>Akka Sample: Plan Approval Gate (HITL)</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — submit a goal; live list of plans via SSE; per-plan Edit, Approve, and Cancel buttons shown only when `status == PLANNED` and `steps` is present. Approved plans show the outcome; cancelled plans show the reason. Editing reveals a textarea for revised steps and a save button that calls PATCH.

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

- **Goal input** — single text field + Submit button → `POST /api/plan-request`.
- **Plans list** — one card per plan: goal, status badge, numbered steps (collapsed after 3 items with a "show more" toggle), rationale, and (for `PLANNED` plans) an Edit button, an approver-name field, a note field, Approve and Cancel buttons. Cancel reveals a reason field.
- **Edit mode** — clicking Edit on a `PLANNED` card reveals a textarea pre-filled with the current steps (one per line), an "edited by" field, and a Save button that calls `PATCH /api/plans/{id}/edit`.
- **Status badge** — `PLANNED` neutral, `APPROVED` yellow, `EXECUTED` green, `CANCELLED` muted red.
