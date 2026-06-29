# UI mockup — fomc-event-analyst

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: FOMC Event Analyst</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `FOMC Event <span class="accent">Analyst</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded FOMC events (or type your own event identifier) and click **Run analysis**.
  3. Watch the card transition through GATHERING → GATHERED → INTERPRETING → INTERPRETED → SYNTHESIZING → SYNTHESIZED → REVIEWED.
  4. Inspect the review-log strip on the card if the financial output guardrail fired a rejection.
- Card **How it works**: one paragraph on the three task phases (GATHER → INTERPRET → SYNTHESIZE) and the typed handoff between them; one paragraph on the financial output guardrail (before-agent-response, quality checks, retry path).
- Card **Components**: rows per component (FomcEventEntity, FomcPipelineWorkflow, FomcAnalystAgent, GatherTools, InterpretTools, SynthesizeTools, FinancialOutputGuardrail, PolicyAnalysisView, FomcEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the event identifier is the only user input; indicators come from an in-process corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an event. <span class="accent">Read the analysis.</span>`. Subtitle: `One agent, three task phases, one output review gate.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `FOMC Event` (with a "Pick a seeded event" dropdown that fills it), and a yellow `Run analysis` button.
    - Live list below: one card per analysis, newest-first. Each card shows status pill, guardrail verdict badge (ACCEPTED / REJECTED-AND-RETRIED when `REVIEWED`), event identifier, age, and a small red dot if any guardrail rejection fired.
  - **Right column** — Selected-analysis detail.
    - Header: status pill + verdict badge + event identifier.
    - Phase panel 1 (Market Snapshot): a table with columns indicator, value, unit, observed. Visible once `snapshot` is present.
    - Phase panel 2 (Policy Signals): a list of signals with direction chip (hawkish=red / dovish=green / neutral=grey), magnitude, rationale, and grounded-indicator reference. Visible once `signalSet` is present.
    - Phase panel 3 (Policy Analysis): event name, rate outlook chip, executive summary paragraph, then a per-section block (heading, body, grounding refs as small chips). Visible once `analysis` is present.
    - Forecast panel at bottom: expected rate move (basis points), confidence bar, narrative. Visible when analysis is present.
    - Review-log strip (always visible when `review` is present): shows verdict badge, reason, timestamp, and any prior rejections in chronological order.
- Status pill colours: CREATED=muted, GATHERING=blue, GATHERED=blue, INTERPRETING=yellow, INTERPRETED=yellow, SYNTHESIZING=blue, SYNTHESIZED=blue, REVIEWED=green, FAILED=red.
- Verdict badge colours: ACCEPTED=green, REJECTED-AND-RETRIED=amber.

Each phase panel renders only when its data is present on the row record. An analysis in `INTERPRETING` shows panels 1 and 2 (panel 2 with a spinner until the INTERPRET task returns). This is the visual proof that the typed handoff between phases is the only path information travels.
