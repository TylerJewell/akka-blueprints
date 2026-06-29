# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark, yellow accent, Instrument Sans, dot-grid background. Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

Browser title: `<title>Akka Sample: AI Security Triage Agent</title>`.

Five left-nav tabs. Each `.nav-tab` carries `data-tab="N"`; each `.tab-panel` carries `data-panel="N"`.

## Tab 1 — Overview

Eyebrow "Overview" + headline "AI Security Triage Agent", **no subtitle**. Four cards:

- **Try it** — the single command `/akka:build`. No env-var export block.
- **How it works** — three lines: decompose the incident signal, parallel fan-out to VulnerabilityScanner and ThreatContextAgent, merge and triage with a security officer approval gate before any mitigation executes.
- **Components** — the count: 3 autonomous-agent, 1 workflow, 3 event-sourced-entity, 1 view, 1 consumer, 2 timed-action, 2 http-endpoint.
- **API contract** — the top-level endpoints from `api-contract.md`.

## Tab 2 — Architecture

The four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER) plus a click-to-expand component table with syntax-highlighted Java snippets. Carry the Lesson 24 CSS overrides: white `fill`/`color` on every state-label DOM path, `overflow:visible` on edge-label `foreignObject`, and `transitionLabelColor:#cccccc` in `mermaid.initialize`.

## Tab 3 — Risk Survey

Renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style from `governance.html`. Question on the left (yellow label), answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Renders `/api/metadata/eval-matrix` in `matrix-card` / `matrix-row` style. The label column carries the control id and a colored mechanism pill (`hitl` orange, `eval-event` blue). Click a row to expand name, rationale, and implementation. Two controls: H1, E1.

## Tab 5 — App UI

A form with fields for `assetId`, `assetType`, `signalDescription`, `reportedBy`, and `initialSeverity` (dropdown: LOW / MEDIUM / HIGH / CRITICAL) and a Submit button (`POST /api/incidents`). Below the form, a live list of incidents streamed from `/api/incidents/sse`. Each row shows the asset ID, signal description (truncated), a status pill (RECEIVED / TRIAGING / AWAITING_APPROVAL / MITIGATED / REJECTED / DEGRADED), the severity, and the eval score when present.

Incidents in `AWAITING_APPROVAL` show inline **Approve** and **Reject** buttons. Clicking Approve opens a small inline form for `officerId` and an optional `reason`; clicking Reject requires `officerId` and `reason`. On submit the buttons call `POST /api/incidents/{id}/approve` or `.../reject` and the row updates via SSE.

Click a row to expand the full `vulnerabilities` list (table: cveId / cvssScore / component / source), the `threatContext` actors (table: group / attack pattern / precedent), the triage report summary and mitigation plan, the approval decision (officer + reason + timestamp), and the eval score + rationale.

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
