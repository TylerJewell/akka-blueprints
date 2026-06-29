# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchor: `specs/vision/governance.html`). Browser title: `<title>Akka Sample: Support Agent with HITL</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `sanitizer` blue, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — submit a ticket subject and body; live list of tickets via SSE; per-ticket Approve/Escalate buttons shown only when `status == TRIAGED` and `triageSummary` is present. Resolved tickets show the resolution text; escalated tickets show the escalation reason.

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

- **Subject input** — single text field for the ticket subject.
- **Body input** — multi-line textarea for the ticket body text.
- **Submit button** — `POST /api/tickets`.
- **Tickets list** — one card per ticket: category and priority badges, triage summary, and (for `TRIAGED` tickets) an approver-name field, a note field, Approve and Escalate buttons. Escalate reveals a reason field.
- **Status badge** — `TRIAGED` neutral, `APPROVED` yellow, `RESOLVED` green, `ESCALATED` muted red.
- **Resolution pane** — for `RESOLVED` tickets, shows the customer-facing response text.
