# UI mockup — tau2-benchmark-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Tau2 Benchmark Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Tau2 <span class="accent">Benchmark Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a task category (web-navigation / tool-use / multi-step-reasoning) or paste a custom task definition.
  3. Click **Run task**.
  4. Watch the card transition through EXECUTING → RESULT_RECORDED → SCORED and the aggregate panel update.
- Card **How it works**: one paragraph on submit → execute → score; one paragraph on the on-decision evaluator (performance monitor, rule-based, no LLM).
- Card **Components**: rows per component (BenchmarkRunEntity, BenchmarkRunWorkflow, BenchmarkAgent, TaskScorer, BenchmarkView, BenchmarkEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Scorer as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data and `decisions.authority_level = automated` are the distinctive answers for this general benchmark domain. `oversight.human_in_loop = false` and `oversight.human_on_loop = true` signal that humans inspect aggregates, not individual runs. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (E1). ID badge coloured blue (eval-periodic).
- Row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">See the score.</span>`. Subtitle: `One agent, one performance monitor around it.`
- Layout: two-column.
  - **Left column** — Aggregate panel + submission panel + live list.
    - Aggregate panel at top: pass rate gauge (percentage), mean score (1–5), per-category bar (web-navigation / tool-use / multi-step-reasoning pass counts). Updates in real time as SSE events arrive.
    - Submission panel: dropdown `Task category` (web-navigation / tool-use / multi-step-reasoning / custom), `Task ID` text input (auto-filled when a seeded category is picked), `Description` textarea, `Steps` textarea (newline-separated), `Reference answer` text input, `Submitted by` text input, and a yellow `Run task` button. A "Load seeded example" link fills all fields from the seeded JSONL.
    - Live list below: one card per run, newest-first. Each card shows status pill, outcome badge (when result landed), score chip (when scored), task id, category chip, age.
  - **Right column** — Selected-run detail.
    - Header: status pill + outcome badge + score chip + task id + category chip.
    - Task description: the `task.description` string.
    - Steps: an ordered list of step descriptions, each followed by the corresponding `StepOutput.output` in a monospace block. An `attempted = false` step is highlighted with a red left border.
    - Final answer: the agent's `finalAnswer` in a highlighted box; a tick or cross indicating whether it matches the reference answer.
    - Reference answer: the `task.referenceAnswer` in muted text.
    - Latency: `{latencyMs} ms` displayed as a pill.
    - Score section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, EXECUTING=yellow, RESULT_RECORDED=blue, SCORED=green, FAILED=red.
- Outcome badge colours: PASS=green, PARTIAL=yellow, FAIL=red.
- Score chip colours: 5=green, 4=teal, 3=yellow, 2=orange, 1=red.
