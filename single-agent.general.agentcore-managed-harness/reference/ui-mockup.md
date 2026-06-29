# UI mockup — managed-harness-tool-use-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Managed-Harness Tool-Use Agent</title>`.

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
- Headline: `Managed-Harness Tool-Use <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick an agent profile (ops-read-only / ops-read-write / analytics-only) and pick a seeded question or type your own.
  3. Click **Ask**.
  4. Watch the card transition through RUNNING → ANSWERED, then expand the tool-call trace.
- Card **How it works**: one paragraph on submit → tool loop → answer; one paragraph on the before-tool-invocation guardrail and what happens when the agent requests a blocked tool.
- Card **Components**: rows per component (QueryEntity, QueryWorkflow, OperationsAgent, ToolAllowlistGuardrail, AgentProfiles, ToolStubs, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 2 supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `sector: operations` declaration in Purpose is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the answer.</span>`. Subtitle: `One agent, a managed tool loop, one guardrail around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Agent profile` (ops-read-only / ops-read-write / analytics-only), a text area labelled `Question` pre-populated with a "Load seeded example" link, `Requested by` text input, and a yellow `Ask` button.
    - Below the submission panel: a small profile-info card showing the selected profile's permitted tools as chips.
    - Live list below: one card per query, newest-first. Each card shows status pill, question excerpt (first 80 chars), profile badge, age.
  - **Right column** — Selected-query detail.
    - Header: status pill + profile badge + `queryId`.
    - Question: the full question text.
    - Prose answer: rendered once the card reaches `ANSWERED`.
    - Tool-call trace: an ordered table. Columns: #, tool name (coloured chip), arguments summary, output summary, blocked indicator (⛔ if `blocked=true`). Rows appear as the answer is received (not streamed in real time — the full trace arrives with the `ANSWERED` SSE event).
    - Summary chips at bottom: total tool calls, blocked calls.
- Status pill colours: SUBMITTED=muted, RUNNING=yellow, ANSWERED=green, FAILED=red.
- Tool-name chip colours: `query_metrics`=blue, `list_resources`=teal, `fetch_logs`=purple, `compute_stats`=orange, blocked call=red.
- Profile badge colours: ops-read-only=muted, ops-read-write=blue, analytics-only=teal.
