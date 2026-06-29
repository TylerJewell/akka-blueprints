# UI mockup — durable-workflow-backed-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Durable Workflow-Backed Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Durable Workflow-Backed <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded incidents (or enter a custom alertId/serviceId) and click **Remediate**.
  3. Watch the card transition through DIAGNOSING → DIAGNOSED → REMEDIATING → REMEDIATED → VERIFYING → VERIFIED.
  4. Trigger a durability report via the **Evaluate now** button and inspect the four-metric tile row.
- Card **How it works**: one paragraph on the three task phases (DIAGNOSE → REMEDIATE → VERIFY) and the workflow's durability guarantees (step timeouts, retry policy, failover step); one paragraph on the two governance mechanisms (budget-cap guardrail, scheduled durability evaluator).
- Card **Components**: rows per component (IncidentEntity, DurabilityReportEntity, RemediationWorkflow, OpsAgent, DiagnoseTools, RemediateTools, VerifyTools, BudgetGuardrail, DurabilityEvaluator, DurabilityTimerAction, IncidentView, RemediationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.
- **Durability report panel** at the bottom of the Overview tab: four metric tiles (Completion rate, Budget-violation rate, MTTR, Timeout rate) loaded from `GET /api/durability/latest`. A small **Evaluate now** button posts to `POST /api/durability/trigger` and refreshes the tiles. If no report has been generated yet, tiles show `—`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, 1 Evaluator, 1 TimerAction as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (incidents are service-level, not person-level). `decisions.authority_level = automated-action` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, E1). ID badges coloured: H1 red (halt guardrail), E1 blue (eval-periodic).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an incident. <span class="accent">Watch it resolve.</span>`. Subtitle: `One agent, three task phases, one budget cap between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Alert ID` and text input `Service ID` (with a "Pick a seeded incident" dropdown that fills both), and a yellow `Remediate` button.
    - Live list below: one card per incident, newest-first. Each card shows status pill, alertId, age, and a small red dot if any budget violation fired during this incident.
  - **Right column** — Selected-incident detail.
    - Header: status pill + alertId + serviceId.
    - Phase panel 1 (Diagnosis): root cause heading, severity badge, evidence-metrics table (metricName, value, unit, sampledAt), evidence-logs table (level, message, timestamp). Visible once `diagnosis.isPresent()`.
    - Phase panel 2 (Remediation): actions-applied list (action, target, receiptId, outcome chip). Visible once `remediation.isPresent()`.
    - Phase panel 3 (Verification): health-checks table (serviceId, healthy chip, latencyP99Ms), resolution-check row (resolved chip, evidence text). Visible once `verification.isPresent()`.
    - Budget-violation strip (only visible if `budgetViolations` is non-empty): a small table with phase, tool, callsUsed/cap, violatedAt.
- Status pill colours: CREATED=muted, DIAGNOSING=blue, DIAGNOSED=blue, REMEDIATING=yellow, REMEDIATED=yellow, VERIFYING=blue, VERIFIED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. An incident in `REMEDIATING` shows panels 1 and 2 (panel 2 with a spinner if the agent has not yet returned the `RemediationPlan`). This is the visual proof that the typed handoff between phases is the only path information travels.
