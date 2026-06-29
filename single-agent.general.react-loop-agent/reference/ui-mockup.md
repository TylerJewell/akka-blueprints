# UI mockup — react-loop-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: ReAct Agent Workflow</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index.

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `ReAct Agent <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded query or type your own.
  3. Select the tools to enable from the checklist.
  4. Click **Run** and watch the step trace fill in: Thought → Action → Observation → ... → Answer.
- Card **How it works**: one paragraph on query submission → agent loop → step recording; one paragraph on the two governance mechanisms (tool-call guardrail, on-decision chain evaluator).
- Card **Components**: rows per component (RunEntity, ToolDispatcher, ReActWorkflow, ReActAgent, ToolCallGuardrail, ChainEvaluator, RunView, RunEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, 1 Guardrail + 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `decisions.authority_level = recommend-only` and `operations.agent_count = 1` / `agent_pattern = single-agent` are the distinctive filled answers. Many deployer-specific fields are `TO_BE_COMPLETED_BY_DEPLOYER` and displayed in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">Follow the reasoning.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded query` (Arithmetic / Research / Conditional lookup / custom), `Query` textarea (pre-filled when seeded query selected), tool checklist (Calculator, WebLookup, DataFetch each with an enable/disable toggle), `Submitted by` text input, and a yellow `Run` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, outcome badge (when result landed), chain score chip (when eval landed), query excerpt (first 60 chars), age.
  - **Right column** — Selected-run detail.
    - Header: status pill + outcome badge + chain score chip + query excerpt.
    - Available tools row: chip per tool (enabled = accent; denied = red).
    - Step trace: a numbered, chronological list of Step cards. Each card shows:
      - THOUGHT: gray chip + italic text.
      - ACTION: blue chip + `toolName` badge + params JSON in a monospace block.
      - OBSERVATION: green chip (or red "[BLOCKED]" chip for blocked steps) + content text.
      - ANSWER: yellow chip + final answer text in a highlighted block.
    - Chain eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: PENDING=muted, RUNNING=yellow (pulsing), COMPLETED=green, EXHAUSTED=orange, FAILED=red.
- Outcome badge colours: ANSWERED=green, EXHAUSTED=orange.
- Step chip colours: THOUGHT=gray, ACTION=blue, OBSERVATION=green, ANSWER=yellow, BLOCKED=red.

The `steps` array is streamed live: each `StepRecorded` SSE event appends a new card to the trace without a page reload. The user sees the agent's reasoning unfold in real time.
