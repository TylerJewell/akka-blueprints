# UI mockup — code-interpreter-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Code Interpreter Agent</title>`.

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
- Headline: `Code Interpreter <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded dataset (CSV sales table / JSON latency metrics / Numeric response times) and enter a prompt, or load both with one click.
  3. Click **Run**.
  4. Watch the job card transition through CODE_APPROVED → EXECUTING → RESULT_RECORDED.
- Card **How it works**: one paragraph on prompt → code-generation → guard → execute → result; one paragraph on the two governance controls (before-tool-call guardrail, automatic safety halt).
- Card **Components**: rows per component (JobEntity, InterpretationWorkflow, CodeInterpreterAgent, CodeGuardrail, ExecutionSandbox, JobView, JobEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 1 Sandbox as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `decisions.authority_level = automated-execution` and `oversight.human_in_loop = false` are the distinctive answers — code runs without per-job human approval. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic. The `failure_modes` list (malicious-code-generation, runaway-execution, forbidden-import-bypass, incorrect-computation, out-of-scope-file-write) is fully rendered.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 orange (halt).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a prompt. <span class="accent">See the result.</span>`. Subtitle: `One agent. Code generation with guardrail + halt.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Dataset format` (CSV / JSON / Numeric), `Prompt` textarea (with placeholder "e.g. Find the top 5 values in the revenue column and compute their mean."), `Data payload` textarea (with a "Load seeded example" link that fills both prompt and data), `Submitted by` text input, and a yellow `Run` button.
    - Live list below: one card per job, newest-first. Each card shows status pill, result-kind chip (when result landed), data-format chip, prompt excerpt (truncated at 60 chars), age.
  - **Right column** — Selected-job detail.
    - Header: status pill + result-kind chip + data-format chip + prompt excerpt.
    - Generated code section: a monospace block showing `pythonSource` with import-list chips above (`csv`, `statistics`, etc.).
    - Execution result: for SCALAR results, a large `answer` value. For TABLE results, a compact HTML table with `columnNames` as headers and `rows` as body. For ERROR results, a red error message block.
    - Wall-clock duration chip: `<N> ms`.
    - Halt section (visible only when `status == HALTED`): red banner showing breach type (`TIMED_OUT` / `MEMORY_EXCEEDED`) and elapsed milliseconds.
- Status pill colours: SUBMITTED=muted, CODE_APPROVED=blue, EXECUTING=yellow, RESULT_RECORDED=green, HALTED=red, FAILED=red.
- Result-kind chip colours: SCALAR=green, TABLE=blue, ERROR=red.
- Data-format chip colours: CSV=muted, JSON=blue, NUMERIC=muted.

The raw data payload is not shown in the left-rail card list — only in the right-pane detail of the selected job. This keeps the list scannable for users with many jobs queued.
