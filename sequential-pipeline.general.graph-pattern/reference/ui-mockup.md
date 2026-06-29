# UI mockup — graph-pattern

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Graph Pattern</title>`.

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
- Headline: `Graph <span class="accent">Pattern</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded tasks (or type your own) and click **Run graph**.
  3. Watch the card transition through PARSING → PARSED → PLANNING → PLANNED → EXECUTING → EXECUTED → MERGING → MERGED → EVALUATED.
  4. Inspect the dependency-violation log strip on the card if any guardrail rejections fired.
- Card **How it works**: one paragraph on the four task phases (PARSE → PLAN → EXECUTE → MERGE) and the typed handoff between them; one paragraph on the graph primitive — nodes in topological order, predecessor-gated execution — and the two governance mechanisms (dependency guardrail, coverage eval).
- Card **Components**: rows per component (GraphRunEntity, GraphExecutionWorkflow, GraphAgent, ParseTools, PlanTools, ExecuteTools, MergeTools, DependencyGuardrail, CoverageScorer, GraphRunView, GraphEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 4 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the task description is the only user input; node outputs come from an in-process corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- The `controls` list is empty for this baseline — display a single message: `No deployer-declared controls. The runtime wires dependency gating and coverage scoring as built-in mechanisms; see Architecture for details.`

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Describe a task. <span class="accent">See the graph run.</span>`. Subtitle: `One agent, four task phases, one runtime gate between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Task description` (with a "Pick a seeded task" dropdown that fills it), and a yellow `Run graph` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, eval score chip (when eval landed), description (truncated to 60 chars), age, and a small red dot if any dependency violation fired during this run.
  - **Right column** — Selected-run detail.
    - Header: status pill + eval score chip + description.
    - Phase panel 1 (Parsed request): intent text and a constraints tag list. Visible once `parsedRequest` is present.
    - Phase panel 2 (Graph plan): a two-column table — Nodes (nodeId, label, predecessorIds) and Edges (from → to). Visible once `plan` is present.
    - Phase panel 3 (Execution): a per-node status list showing each node's status chip (PENDING / RUNNING / DONE), outputId, and body (collapsed by default, expand on click). Visible once `status ∈ {EXECUTING, EXECUTED, MERGING, MERGED, EVALUATED}`.
    - Phase panel 4 (Result): title, summary paragraph, then a per-ref block (nodeId, outputId, a short body excerpt). Visible once `taskResult` is present.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Dependency-violation log strip (only visible if the run has any `dependencyViolations`): a small table with phase, nodeId, missingPredecessors, reason, time.
- Status pill colours: CREATED=muted, PARSING=blue, PARSED=blue, PLANNING=yellow, PLANNED=yellow, EXECUTING=blue, EXECUTED=blue, MERGING=yellow, MERGED=yellow, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A run in `EXECUTING` shows panels 1 and 2 fully, and panel 3 with live per-node status chips updating via SSE `node-executed` events. This is the visual proof that the typed handoff between phases is the only path information travels, and that node-level dependency gating happens at the entity boundary.
