# UI mockup — policy-as-code

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: PolicyAsCode</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The canonical implementation:

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
- Headline: `Policy<span class="accent">AsCode</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded policy set (Infrastructure / Kubernetes / Supply-chain) and paste the matching seed change payload, or load both with one click.
  3. Click **Submit for evaluation**.
  4. Watch the card transition through VALIDATED → ENFORCING → DECISION_RECORDED → GATED. Note the CI gate chip (OPEN / BLOCKED).
- Card **How it works**: one paragraph on submit → validate → enforce → gate; one paragraph on the two governance mechanisms (before-tool-call guardrail, CI gate).
- Card **Components**: rows per component (ChangeRequestEntity, ChangeValidator, EvaluationWorkflow, PolicyEnforcementAgent, ToolCallGuardrail, GateEvaluator, PolicyView, PolicyEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`, including the gate endpoint.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Gate Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = enforce` declaration is filled and prominent. `oversight.human_in_loop = false` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, C1). ID badges coloured: G1 red (guardrail), C1 orange (ci-gate).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a change. <span class="accent">See the gate.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Policy set` (Infrastructure / Kubernetes / Supply-chain / custom), `Change title` text input, `Change type` dropdown (terraform-plan / k8s-manifest / dependency-bump / custom), `Change payload` textarea (with a "Load seeded example" link that fills title, type, and payload), `Submitted by` text input, and a yellow `Submit for evaluation` button.
    - Live list below: one card per evaluation, newest-first. Each card shows status pill, outcome badge (when decision landed), gate chip (OPEN / BLOCKED when gate landed), change title, age.
  - **Right column** — Selected-evaluation detail.
    - Header: status pill + outcome badge + gate chip + `changeTitle`.
    - Submitted policy rules: a small list with `ruleId`, `category` chip, `severityFloor` chip, and the rule text.
    - Validated change preview: a monospace block of the normalized payload, with affected-resource chips above.
    - Decision rationale: the agent's 1–3-sentence paragraph.
    - Violations table: columns rule id, severity (coloured chip), affected resource, evidence (monospace), recommendation.
    - CI gate section at bottom: OPEN (green) or BLOCKED (red) badge plus the gate reason string. BLOCKED highlights the card border red.
- Status pill colours: SUBMITTED=muted, VALIDATED=blue, ENFORCING=yellow, DECISION_RECORDED=blue, GATED=green, FAILED=red.
- Outcome badge colours: ALLOW=green, WARN=yellow, DENY=red.
- Gate chip colours: OPEN=green, BLOCKED=red.
- Severity chip colours: LOW=muted, MEDIUM=blue, HIGH=yellow, CRITICAL=red.

The raw change payload is never displayed on this screen — only the normalized form. Operators who need the raw payload fetch `/api/changes/{id}` and read `request.rawPayload` from the JSON. This is intentional: the UI shows what the agent saw, even though the audit trail keeps the raw.
