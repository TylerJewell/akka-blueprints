# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchor: `specs/vision/governance.html`). Browser title: `<title>Akka Sample: HITL Tool Approval</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — submit a goal; live list of tool requests via SSE; per-request Approve/Reject buttons shown only when `status == PENDING_APPROVAL` and `toolName` is present. Executed requests show the output; rejected requests show the reason.

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

- **Goal input** — single text field + Submit button → `POST /api/tool-requests`.
- **Requests list** — one card per request: tool name, status badge, parameters preview, rationale, and (for `PENDING_APPROVAL` requests) an approver-name field, an optional edited-parameters field, an approver-note field, Approve and Reject buttons. Reject reveals a reason field.
- **Edited-parameters field** — pre-populated with the planner's `parameters` value; operator may edit before approving. Sent as `editedParameters` in the approve body only when the value differs from the original.
- **Status badge** — `PENDING_APPROVAL` neutral, `APPROVED` yellow, `EXECUTED` green, `REJECTED` muted red.
- **Executed output** — shown below the parameters on `EXECUTED` requests; a scrollable pre-formatted block.
