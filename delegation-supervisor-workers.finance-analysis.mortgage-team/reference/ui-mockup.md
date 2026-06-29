# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark, yellow accent, Instrument Sans, dot-grid background. Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

Browser title: `<title>Akka Sample: Mortgage Assistant (Multi-Agent)</title>`.

Five left-nav tabs. Each `.nav-tab` carries `data-tab="N"`; each `.tab-panel` carries `data-panel="N"`.

## Tab 1 — Overview

Eyebrow "Overview" + headline "Mortgage Assistant (Multi-Agent)", **no subtitle**. Four cards:

- **Try it** — the single command `/akka:build`. No env-var export block.
- **How it works** — three lines: decompose the application, parallel fan-out to EligibilityAgent + DocumentationAgent + UnderwritingAgent, merge assessments then wait for underwriter sign-off.
- **Components** — the count: 5 autonomous-agent, 1 workflow, 2 event-sourced-entity, 1 view, 1 consumer, 2 timed-action, 2 http-endpoint.
- **API contract** — the top-level endpoints from `api-contract.md`.

## Tab 2 — Architecture

The four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER) plus a click-to-expand component table with syntax-highlighted Java snippets. Carry the Lesson 24 CSS overrides: white `fill`/`color` on every state-label DOM path, `overflow:visible` on edge-label `foreignObject`, and `transitionLabelColor:#cccccc` in `mermaid.initialize`.

## Tab 3 — Risk Survey

Renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style from `governance.html`. Question on the left (yellow label), answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Renders `/api/metadata/eval-matrix` in `matrix-card` / `matrix-row` style. The label column carries the control id and a colored mechanism pill (`sanitizer` orange, `hitl` green, `eval-event` blue). Click a row to expand name, rationale, and implementation. Four controls: S1, S2, H1, E1.

## Tab 5 — App UI

A form with inputs for applicant reference, loan product (select: 30-year-fixed, 15-year-fixed, 5-1-ARM), requested amount (GBP), property value (GBP), employment status (select: FULL_TIME, PART_TIME, SELF_EMPLOYED, CONTRACT), annual income (GBP), credit score, and document IDs (comma-separated), plus a Submit button (`POST /api/applications`). Below it, a live list of applications streamed from `/api/applications/sse`. Each row shows the applicant reference, loan product, status pill (RECEIVED / UNDER_REVIEW / PENDING_SIGNOFF / APPROVED / DECLINED / COMPLIANCE_HOLD / REVIEW_REQUIRED), and eval score when present. Click a row to expand:
- Eligibility verdict (eligible boolean, LTV, DTI, verdict text)
- Documentation assessment (complete boolean, missing document list, authenticity flags)
- Underwriting assessment (risk band pill, proposed rate, amortisation, conditions, rationale)
- Compliance status
- Sanitised summary from the supervisor
- Eval score and rationale
- **HITL decision panel** (shown only when status is `PENDING_SIGNOFF`): two buttons — Approve / Decline — with a notes text field and decidedBy input. Submits to `POST /api/applications/{id}/decision`.

## Tab switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` / `data-panel` attribute, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Exactly five `.tab-panel` sections exist in the DOM. Removed tabs are deleted, never hidden with `display:none` — a hidden zombie panel takes an index slot and blanks the App UI tab.
