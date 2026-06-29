# UI mockup — async-agent-endpoint

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: AsyncAgentEndpoint</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The exemplar's `static-resources/index.html` has the canonical implementation:

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
- Headline: `Async Agent <span class="accent">Endpoint</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded task from the dropdown (vowel counter / CSV average / Fibonacci) or type a custom prompt.
  3. Click **Run task**.
  4. Watch the card transition through RUNNING → COMPLETED (or FAILED if the guardrail exhausts its retries).
- Card **How it works**: one paragraph on submit → run → record; one paragraph on the guardrail mechanism (before-tool-call sandbox policy check).
- Card **Components**: rows per component (AgentRunEntity, AgentRunWorkflow, CodeRunnerAgent, SandboxGuardrail, RunView, RunEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `code_execution_sandboxed: true` declaration in Data is filled and prominent. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Read the result.</span>`. Subtitle: `One agent, one sandbox guardrail.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Task` (vowel counter / CSV average / Fibonacci / custom), `Task prompt` textarea (pre-filled when a seeded task is selected), `Submitted by` text input, and a yellow `Run task` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, guardrail-hit badge (if `guardrailHitOnAnyIteration` is true), prompt excerpt, age.
  - **Right column** — Selected-run detail.
    - Header: status pill + guardrail-hit badge (if applicable) + prompt excerpt.
    - Task prompt: the full `promptText` in a muted block.
    - Generated code: a monospace code block with syntax highlighting (language: Python).
    - Execution output: stdout in a monospace block, stderr in a muted red monospace block (if non-empty), output value as a highlighted badge.
    - Failure detail (if `FAILED`): `FailureKind` badge + message.
- Status pill colours: SUBMITTED=muted, RUNNING=yellow, COMPLETED=green, FAILED=red.
- Guardrail-hit badge: amber, shown only when `guardrailHitOnAnyIteration = true` on a `COMPLETED` run (informational — the run succeeded, but a prior iteration was rejected).
- FailureKind badge colours: GUARDRAIL_EXHAUSTED=red, EXECUTION_ERROR=orange, TIMEOUT=yellow.
