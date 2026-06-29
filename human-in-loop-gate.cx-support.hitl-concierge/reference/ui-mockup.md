# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchor: `specs/vision/governance.html`). Browser title: `<title>Akka Sample: HITL Concierge</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — submit a customer request; live list of cases via SSE; per-case Approve/Decline buttons shown only when `status == TRIAGED` and `proposedResolution` is present. Fulfilled cases show the confirmation id; declined cases show the reason.

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

- **Customer request input** — multi-line text area + Submit button → `POST /api/cases`.
- **Cases list** — one card per case: triage summary, urgency badge, proposed resolution, status badge, and (for `TRIAGED` cases) a specialist-name field, a comment field, Approve and Decline buttons. Decline reveals a reason field.
- **Status badge** — `TRIAGED` neutral, `APPROVED` yellow, `FULFILLED` green, `DECLINED` muted red.
- **Urgency badge** — `HIGH` red, `MEDIUM` orange, `LOW` muted.
