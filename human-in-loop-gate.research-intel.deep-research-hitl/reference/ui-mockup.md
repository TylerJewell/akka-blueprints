# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchor: `specs/vision/governance.html`). Browser title: `<title>Akka Sample: HITL Deep Research</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — submit a query; live list of reports via SSE; per-report Approve/Reject buttons shown only when `status == AWAITING_REVIEW` and `reportBody` is present. Delivered reports show the delivery timestamp; `NEEDS_REVISION` reports show the reviewer's notes. Status progression is shown for in-flight reports (PLANNING → INVESTIGATING → SYNTHESISING → AWAITING_REVIEW).

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

- **Query input** — single text field + Submit button → `POST /api/research-request`.
- **Reports list** — one card per report: title (or query as fallback), status badge, sub-topics list (when INVESTIGATING), report body preview (when AWAITING_REVIEW or later), and (for `AWAITING_REVIEW` reports) a reviewer-name field, a notes field, Approve and Reject buttons.
- **Status badge** — `PLANNING` neutral-grey, `INVESTIGATING` blue, `SYNTHESISING` purple, `AWAITING_REVIEW` yellow, `APPROVED` green, `NEEDS_REVISION` orange, `DELIVERED` green-bright.
- **In-progress indicator** — for PLANNING, INVESTIGATING, SYNTHESISING statuses, show a pulsing dot next to the badge to indicate background work is running.
