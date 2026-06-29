# UI mockup — akka-resumable-agent-http

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: DurableAgentHTTP</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Durable<span class="accent">AgentHTTP</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Enter a location (or pick a seeded example) and click **Run agent**.
  3. While the status shows `TOOL_CALLED`, kill the process with Ctrl-C.
  4. Restart via `/akka:build` and watch the run resume from its checkpoint.
- Card **How it works**: one paragraph on request → initStep → toolCallStep (slow tool, kill window) → reportStep; one paragraph on the two governance mechanisms (before-tool-call guardrail, on-incident evaluator).
- Card **Components**: rows per component (AgentRunEntity, AgentRunWorkflow, WeatherQueryAgent, SlowWeatherTool, ToolCallGuardrail, IncidentEvaluator, AgentRunView, AgentEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `crash_resume_enabled: true` and `checkpoint_granularity: per-workflow-step` declarations are filled and prominent. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Run the agent. <span class="accent">Kill it. Watch it resume.</span>`. Subtitle: `One agent, two governance mechanisms, durable checkpoints.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Query type` (CURRENT / FORECAST / AIR_QUALITY), `Location` text input (with a "Use seeded example" link that fills it), `Requested by` text input, and a yellow **Run agent** button.
    - Live list below: one card per run, newest-first. Each card shows status pill, resume badge (when `resumeCount > 0`), guardrail chip (when `guardrailRejected == true`), location string, age.
  - **Right column** — Selected-run detail.
    - Header: status pill + resume badge + location + query type.
    - Tool-call status: tool name, argument used, called-at timestamp, guardrail-rejected indicator (if applicable) with rejection-reason chip.
    - Weather report section (visible when report lands): summary paragraph, temperature display (°C), conditions badge.
    - Incident panel at bottom (visible when `incident` is non-null): severity badge, interrupted step, elapsed-before-crash, notes text.
- Status pill colours: INITIATED=muted, TOOL_CALLED=yellow (animated), REPORTING=blue, RESUMED=orange, COMPLETED=green, FAILED=red.
- Resume badge: orange `Resumed ×N` where N is `resumeCount`.
- Guardrail chip: red `Guardrail` with tooltip showing `rejectionReason`.
- IncidentSeverity badge colours: INFO=muted, WARNING=yellow, ERROR=red.
- Conditions badge: green for sunny/clear, blue for rain/fog, grey for unknown.
