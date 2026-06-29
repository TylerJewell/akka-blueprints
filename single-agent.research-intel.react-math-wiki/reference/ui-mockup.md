# UI mockup — react-math-wiki

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: MathWikiReAct</title>`.

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
- Headline: `Math + Wikipedia <span class="accent">ReAct</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded question from the dropdown, or type your own.
  3. Click **Ask**.
  4. Watch the card update in real time as the agent calls tools — step by step — then shows the final answer and eval score.
- Card **How it works**: one paragraph on submit → ReAct loop → answer → eval; one paragraph on the guardrail mechanism (math-expression sandbox on `evaluate_math` before each call).
- Card **Components**: rows per component (ResearchEntity, ResearchWorkflow, ResearchAgent, MathGuardrail, MathEvaluator, WikipediaStub, AnswerEvaluator, ResearchView, ResearchEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 2 supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `pii: false` is prominent. `decisions.authority_level = inform-only` and `oversight.human_in_loop = false` are the distinctive answers. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Watch the agent think.</span>`. Subtitle: `One agent, two tools, one math sandbox.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded questions` (5 options + "Custom"), `Question` textarea (pre-filled when a seeded question is selected), `Submitted by` text input, and a yellow `Ask` button.
    - Live list below: one card per question, newest-first. Each card shows status pill, confidence badge (when answer landed), eval score chip (when eval landed), truncated question text, age, and step count.
  - **Right column** — Selected-question detail.
    - Header: status pill + confidence badge + eval score chip + question text.
    - Tool-call trace: a monospace table that grows in real time via SSE. Columns: step, tool chip (MATH=purple, WIKI=blue), argument, observation. Rows appear as `ToolCallRecorded` events arrive — before the answer is ready.
    - Answer section (visible once `ANSWERED`): `answerText` in a highlighted block, confidence badge, step count.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, RUNNING=yellow, ANSWERED=blue, EVALUATED=green, FAILED=red.
- Confidence badge colours: HIGH=green, MEDIUM=yellow, LOW=red.
- Tool chip colours: EVALUATE_MATH=purple, SEARCH_WIKIPEDIA=blue.
