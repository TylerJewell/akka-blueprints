# UI mockup — hook-instrumentation

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Hook Instrumentation</title>`.

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
- Headline: `Hook <span class="accent">Instrumentation</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded task template (summarize-report / calculate-metrics / lookup-policy) and optionally tick **simulate blocked tool**.
  3. Click **Submit task**.
  4. Watch the card transition through HOOK_CHAIN_READY → EXECUTING → COMPLETED; expand the hook-log table.
- Card **How it works**: one paragraph on submit → hook-chain init → execute → coverage score; one paragraph on the three hook points (before-tool-call, after-tool-call, before-llm-call).
- Card **Components**: rows per component (ObservationEntity, HookLogConsumer, ObservationWorkflow, ActivityObserverAgent, BeforeToolCallGuardrail, AfterToolCallGuardrail, BeforeLlmCallGuardrail, CoverageScorer, ObservationView, ObservationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 3 Guardrails + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `credentials-or-secrets: true` declaration in Data is filled and prominent. `decisions_surface = operational-execution` and `human_on_loop = true` are the distinctive answers. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, G2, G3). ID badges coloured: all three red (guardrail), with hook-point labels below each ID (BEFORE TOOL CALL / AFTER TOOL CALL / BEFORE LLM CALL).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Inspect the hook log.</span>`. Subtitle: `One agent, three hook observers around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Task template` (summarize-report / calculate-metrics / lookup-policy / custom), `Task description` textarea (with a "Load seeded example" link that fills the description and tool set), `Persona` text input (default `observer-standard`), `Simulate blocked tool` checkbox, `Submitted by` text input, and a yellow `Submit task` button.
    - Live list below: one card per observation, newest-first. Each card shows status pill, outcome badge (when outcome landed), coverage score chip (when coverage landed), task description excerpt, age.
  - **Right column** — Selected-observation detail.
    - Header: status pill + outcome badge + coverage score chip + task description excerpt.
    - Hook config summary: three small chips for each hook point (BEFORE TOOL CALL / AFTER TOOL CALL / BEFORE LLM CALL) with blocked-tool count and pattern count.
    - Agent outcome text: the agent's 1–5-sentence response paragraph.
    - Hook log table: columns — hook point chip (colour-coded), tool/phase name, verdict chip, sanitized payload (truncated), fired-at timestamp. BLOCKED rows highlighted in red. REDACTED rows highlighted in amber. Clicking a row expands it to show `rejectReason` (if present) and `originalPayloadHash`.
    - Coverage section at bottom: a 1–5 star widget and the one-line note. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, HOOK_CHAIN_READY=blue, EXECUTING=yellow, COMPLETED=green, FAILED=red.
- Outcome badge colours: SUCCESS=green, PARTIAL=yellow, BLOCKED=red.
- Hook verdict chip colours: ALLOWED=muted, PASS_THROUGH=muted, REDACTED=amber, BLOCKED=red.
- Hook point chip colours: BEFORE_TOOL_CALL=blue, AFTER_TOOL_CALL=purple, BEFORE_LLM_CALL=orange.

The raw tool output is never displayed on this screen — only the sanitized form plus the SHA-256 hash. Operators who need the original tool output for forensic review must access the entity audit log directly via `GET /api/observations/{id}` and read `outcome.hookLog[*].originalPayloadHash`.
