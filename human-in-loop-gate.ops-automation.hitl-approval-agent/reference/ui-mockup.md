# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid. Browser title: `<title>Akka Sample: Human-in-the-Loop Approval Agent</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — submit an operation request; live list of actions via SSE; per-action Approve/Deny buttons shown only when `status == PROPOSED` and `rationale` is present. Executed actions show the outcome; denied actions show the reason.

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

- **Operation request input** — single text field + Submit button → `POST /api/action-request`. Placeholder: "e.g. scale web-frontend to 5 replicas".
- **Actions list** — one card per action. Each card shows:
  - Header: action id (truncated), status badge, timestamp.
  - Proposal block (when `PROPOSED`, `APPROVED`, `DENIED`, or `EXECUTED`): Action Type, Target, Rationale, Estimated Impact — rendered as labeled rows.
  - Approval controls (only when `status == PROPOSED` and `rationale` is non-null): approver name field, optional note field, Approve button. Deny button reveals a reason field inline.
  - Execution result block (when `status == EXECUTED`): Outcome, Completed At, Details.
  - Denial block (when `status == DENIED`): Denied by, Reason.
- **Status badge** — `PROPOSED` neutral, `APPROVED` yellow, `EXECUTED` green, `DENIED` muted red.
- **SSE subscription** — opened once on page load against `GET /api/actions/sse`. On each `action` event, the card with matching `id` is updated in place. If no card matches, a new card is prepended.

## Mermaid rendering (Architecture tab)

Load mermaid from CDN. Before calling `mermaid.initialize`, inject the theme variables block:

```js
mermaid.initialize({
  startOnLoad: false,
  theme: 'base',
  themeVariables: {
    primaryColor: '#1f1900',
    primaryTextColor: '#F5C518',
    primaryBorderColor: '#F5C518',
    lineColor: '#888888',
    transitionLabelColor: '#cccccc'
  }
});
```

After rendering, apply CSS overrides via a `<style>` tag injected into the shadow DOM or document head:

```css
.statediagram-state rect { fill: #1f1900; stroke: #F5C518; }
.statediagram-state text { fill: #F5C518 !important; }
.edgeLabel foreignObject { overflow: visible; }
.transition text { fill: #cccccc; }
```

These ensure state labels render with a legible foreground colour against the dark background (Lesson 24).
