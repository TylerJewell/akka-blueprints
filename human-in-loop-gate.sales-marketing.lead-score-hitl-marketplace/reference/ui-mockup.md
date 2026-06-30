# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchor: `specs/vision/governance.html`). Browser title: `<title>Akka Sample: Lead Score Flow</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `guardrail` red, `eval-event` blue). The value column shows name, rationale, and implementation.
5. **App UI** — submit a lead profile (company name + contact email); live list of leads via SSE; per-lead Approve/Reject controls shown only when `status == SCORED` and `scoreRationale` is present. Qualified leads show the verdict and next steps; disqualified leads show the reason.

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

- **Lead profile input** — company name field + contact email field + Submit button → `POST /api/score-request`.
- **Leads list** — one card per lead: company name, contact email, status badge, score (if scored), rationale summary, and (for `SCORED` leads) an approver-name field, an optional note field, Approve and Reject buttons. Reject reveals a reason field.
- **Status badge** — `SCORED` neutral, `APPROVED` yellow, `QUALIFIED` green, `DISQUALIFIED` muted red.
- **Score display** — numeric score (0–100) shown with a confidence badge (`high` / `medium` / `low`) when status is `SCORED` or beyond.
