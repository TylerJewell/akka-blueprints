# UI mockup — aws-ops-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: AWS Ops Assistant</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `AWS Ops<span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded request (EC2 inventory / S3 storage report / EC2 resize) or type your own.
  3. Click **Submit request**.
  4. Watch the card transition through PLANNING → AWAITING_CONFIRMATION (if mutating) → EXECUTING → COMPLETED.
- Card **How it works**: one paragraph on submit → plan → guardrail-check → confirm (if mutating) → execute → report; one paragraph on the three governance mechanisms (resource allow-list guardrail, per-action HITL confirmation, operator halt).
- Card **Components**: rows per component (OpsRequestEntity, ConfirmationConsumer, OpsWorkflow, AwsOpsAgent, ActionGuardrail, AwsMcpStubs, OpsView, OpsEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 MCP stub layer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = execute` and `oversight.mutating_calls_require_per_action_approval = true` declarations are filled and prominent. `data_classes.cloud-resource-configuration = true` is highlighted. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, X1). ID badges coloured: G1 red (guardrail), H1 amber (hitl), X1 orange-red (halt).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a request. <span class="accent">Confirm the mutations.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Request scope` (Read Only / Mutating / Mixed), `Request` textarea (with a "Load seeded example" link that fills the request text and scope), `Context` text input (optional; e.g., `env=production`), `Submitted by` text input, and a yellow `Submit request` button.
    - Live list below: one card per request, newest-first. Each card shows status pill, report-status badge (when report landed), document title (first 60 chars of requestText), age.
  - **Right column** — Selected-request detail.
    - Header: status pill + report-status badge + truncated requestText.
    - Request metadata: scope chip, context value, submitted-by, submitted-at.
    - Planning summary: short paragraph from the agent (shown once PLANNING transitions to the next state).
    - Confirmation cards: one card per pending HITL action (status == AWAITING_CONFIRMATION). Each card shows: AWS service badge, API call name, resource ARN, action kind chip, rationale, and **Approve** / **Decline** buttons. Clicking either POSTs to `/confirm` immediately.
    - Action results table: columns action id, service, API call, kind chip, resource ARN (truncated), outcome chip, response snippet (expandable), duration ms. Visible once EXECUTING begins.
    - Operation report section at bottom: report-status badge (COMPLETED / BLOCKED / HALTED), summary paragraph, completedAt timestamp.
- Status pill colours: SUBMITTED=muted, PLANNING=blue, AWAITING_CONFIRMATION=yellow, EXECUTING=blue, COMPLETED=green, HALTED=orange, FAILED=red.
- Report-status badge colours: COMPLETED=green, BLOCKED=red, HALTED=orange.
- ActionKind chip colours: READ=muted, MUTATING=yellow.
- ActionOutcome chip colours: COMPLETED=green, SKIPPED=muted, BLOCKED=red, FAILED=red.
- When `status == AWAITING_CONFIRMATION`, the confirmation card pulses with a yellow border to attract attention.
- When `status == HALTED`, the entire detail pane shows a red banner: "This request was halted. Partial results below."
