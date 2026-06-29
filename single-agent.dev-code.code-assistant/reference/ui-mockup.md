# UI mockup — code-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: CodeAssistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Code<span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded repository snapshot (Java REST service / Python data pipeline / TypeScript CLI) and load the matching coding task with one click.
  3. Click **Submit task**.
  4. Watch the card transition through ANALYZING → EDIT_PROPOSED → GATE_PASSED (or GATE_FAILED).
- Card **How it works**: one paragraph on submit → analyze → propose → gate; one paragraph on the two governance mechanisms (before-tool-call guardrail, CI test gate).
- Card **Components**: rows per component (EditEntity, CIGate, EditWorkflow, CodeAssistantAgent, ToolCallGuardrail, EditView, EditEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `source-code: true` declaration in Data is filled and prominent. `decisions.authority_level = propose-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, C1). ID badges coloured: G1 red (guardrail), C1 purple (ci-gate).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Review the plan.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Repository snapshot` (Java REST service / Python data pipeline / TypeScript CLI / custom), `Task description` textarea (with a "Load seeded example" link that fills both the snapshot and the task), `Author` text input, and a yellow `Submit task` button.
    - Live list below: one card per edit task, newest-first. Each card shows status pill, confidence badge (when plan landed), gate badge (GATE_PASSED green / GATE_FAILED red when gate ran), project name, age.
  - **Right column** — Selected-task detail.
    - Header: status pill + confidence badge + gate badge + `taskDescription` (first 80 chars).
    - Task description: full text.
    - File changes table: columns file path, change type chip (ADD/MODIFY/DELETE), diff summary. Each row is click-to-expand; expanded view shows a two-column before/after code block.
    - Test command: a monospace block showing the recommended `testCommand`.
    - CI gate section at bottom: GATE_PASSED shows a green badge + `Tests run: N, Failures: 0` summary. GATE_FAILED shows a red badge + the test failure output in a scrollable monospace block.
- Status pill colours: SUBMITTED=muted, ANALYZING=yellow, EDIT_PROPOSED=blue, GATE_PASSED=green, GATE_FAILED=red, FAILED=red.
- Confidence badge colours: HIGH=green, MEDIUM=yellow, LOW=red.
- Change type chip colours: ADD=green, MODIFY=blue, DELETE=red.
