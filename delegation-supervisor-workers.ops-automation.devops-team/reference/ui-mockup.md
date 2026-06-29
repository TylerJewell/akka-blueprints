# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark, yellow accent, Instrument Sans, dot-grid background. Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

Browser title: `<title>Akka Sample: DevOps Multi-Agent</title>`.

Five left-nav tabs. Each `.nav-tab` carries `data-tab="N"`; each `.tab-panel` carries `data-panel="N"`.

## Tab 1 — Overview

Eyebrow "Overview" + headline "DevOps Multi-Agent", **no subtitle**. Four cards:

- **Try it** — the single command `/akka:build`. No env-var export block.
- **How it works** — three lines: parse the change request into three parallel subtasks, fan out to InfraAgent + DeployAgent + ObservabilityAgent, consolidate into a readiness report with guardrail and approval gate.
- **Components** — the count: 4 autonomous-agent, 1 workflow, 3 event-sourced-entity, 1 view, 1 consumer, 2 timed-action, 2 http-endpoint.
- **API contract** — the top-level endpoints from `api-contract.md`.

## Tab 2 — Architecture

The four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER) plus a click-to-expand component table with syntax-highlighted Java snippets. Carry the Lesson 24 CSS overrides: white `fill`/`color` on every state-label DOM path, `overflow:visible` on edge-label `foreignObject`, and `transitionLabelColor:#cccccc` in `mermaid.initialize`.

## Tab 3 — Risk Survey

Renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style. Question on the left (yellow label), answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Renders `/api/metadata/eval-matrix` in `matrix-card` / `matrix-row` style. The label column carries the control id and a colored mechanism pill (`guardrail` red, `hitl` amber, `halt` orange). Click a row to expand name, rationale, and implementation. Three controls: G1, H1, S1.

## Tab 5 — App UI

A form with three inputs: a `targetEnvironment` selector (dev / staging / production), a `changeType` selector (deploy / scale / config-update / rollback), and a `description` text field. A Submit button posts to `POST /api/ops/changes`. Below the form, a live list of change requests streamed from `/api/ops/changes/sse`. Each row shows the change type, target environment, a status pill, and the overall risk level when present. Click a row to expand the infra assessment, deploy assessment, obs assessment, and the consolidated readiness report summary.

For rows in `AWAITING_APPROVAL` status, the expanded area shows an **Approve** button (calls `POST /api/ops/changes/{id}/approve`) and a **Reject** button (calls `POST /api/ops/changes/{id}/reject`), each prompting for a name / reason before sending. A global **Issue Halt** button in the header calls `POST /api/ops/halt` with a reason from a prompt dialog.

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
