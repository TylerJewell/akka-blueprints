# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid — matching `specs/vision/governance.html`. Content wraps in `.wrap{ max-width:1080px }` with no horizontal scroll (Lesson 12). Browser title: `<title>Akka Sample: Email Auto Responder</title>`.

Five tabs in a left nav: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Tab-switching MUST be attribute-based (Lesson 26)

Every nav tab carries `data-tab="<key>"` and every panel carries a matching `data-panel="<key>"`. Switching toggles `.active` by attribute, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM, never hidden with `display:none` — a hidden zombie panel still occupies an index and breaks index-based switching.

## Overview

Eyebrow "Overview" + headline naming the sample type (not the flow), no subtitle. Four cards: Try it (`/akka:build`, no env-var export block), How it works, Components, API contract.

## Architecture

Renders the four mermaid diagrams from `PLAN.md`. The state diagram includes the Lesson 24 CSS overrides: white state labels via the full `g.statediagram-state .label` DOM-path selectors, and `overflow:visible` on every edge-label `foreignObject`. `mermaid.initialize` sets `nodeTextColor`, `stateLabelColor`, and `transitionLabelColor: #cccccc`.

## Risk Survey

Renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style. Each answer is a row: question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Eval Matrix

Renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill: `sanitizer` green (S1), `guardrail` red (G1), `hitl` yellow (H1). The value column shows name, rationale, and implementation notes. Rows expand on click.

## App UI

The live surface. Elements:

- A "New email" form: `fromAddress`, `subject`, `body` fields and a Send-to-inbox button that POSTs `/api/emails`.
- A live list of emails via `/api/emails/sse`, newest first. Each row shows `fromAddress`, `subject`, a status chip, and — once drafted — the redacted body and the reply draft.
- For rows in `AWAITING_REVIEW`: Approve and Reject buttons. Approve POSTs `/api/emails/{id}/approve`; Reject opens a one-field reason prompt and POSTs `/api/emails/{id}/reject`. The buttons disappear once the row leaves `AWAITING_REVIEW`.
- A status chip colour per state: `SENT` green, `AWAITING_REVIEW` yellow, `REJECTED`/`ESCALATED` red, others muted.
