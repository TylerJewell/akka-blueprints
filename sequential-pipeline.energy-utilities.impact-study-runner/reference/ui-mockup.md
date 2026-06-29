# UI mockup — impact-study-runner

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Grid Impact Study Runner</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Grid Impact <span class="accent">Study Runner</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded interconnection requests (or type your own request ID) and click **Run study**.
  3. Watch the card transition through RUNNING_LOAD_FLOW → LOAD_FLOW_DONE → RUNNING_CONTINGENCY → CONTINGENCY_DONE → DRAFTING → REPORT_DRAFTED → AWAITING_APPROVAL.
  4. Click **Approve** or **Reject** to complete the study lifecycle.
- Card **How it works**: one paragraph on the three task phases (LOAD_FLOW → CONTINGENCY → DRAFT) and the typed handoff between them; one paragraph on the two governance mechanisms (on-decision eval, engineer approval gate).
- Card **Components**: rows per component (StudyEntity, StudyPipelineWorkflow, StudyAgent, LoadFlowTools, ContingencyTools, DraftTools, SimulatorScorer, StudyView, StudyEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (request IDs and grid data are not person-level). `decisions.authority_level = engineer-reviewed-recommendation` and `oversight.human_in_loop = true` are the distinctive answers. NERC-reliability-standards and jurisdiction fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, E1). ID badges coloured: H1 amber (human-in-the-loop), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a request. <span class="accent">Approve the findings.</span>`. Subtitle: `One agent, three task phases, one engineer between the pipeline and issuance.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Request ID` (with a "Pick a seeded request" dropdown that fills it), and a yellow `Run study` button.
    - Live list below: one card per study, newest-first. Each card shows status pill, eval score chip (when eval landed), request ID, age, and an amber highlight on the card border when status is `AWAITING_APPROVAL`.
  - **Right column** — Selected-study detail.
    - Header: status pill + eval score chip + request ID.
    - Phase panel 1 (Load Flow): a bus-voltage table (busId, busName, voltageKv, voltagePu) and a line-flow table (lineId, fromBus, toBus, flowMw, loadPercent). Visible once `loadFlow` is present.
    - Phase panel 2 (Contingency Analysis): a violations table (contingencyId, elementName, limitType, limit, observed — binding violations highlighted red) and a voltage-check table. Visible once `contingency` is present.
    - Phase panel 3 (Study Report): executive summary paragraph, then per-section blocks (heading, body, busRefs chips) and the `meetsN1Criterion` badge (green/red). Visible once `report` is present.
    - Eval section: a 1–5 score chip and the one-line rationale. Score ≤ 2 adds a red border to the approval card.
    - Approval panel (only visible when `status == AWAITING_APPROVAL`): an **Approve** button (green) with an optional note textarea, and a **Reject** button (orange) with a required reason textarea. Submitting either calls the matching endpoint and removes the approval panel.
- Status pill colours: CREATED=muted, RUNNING_LOAD_FLOW=blue, LOAD_FLOW_DONE=blue, RUNNING_CONTINGENCY=yellow, CONTINGENCY_DONE=yellow, DRAFTING=blue, REPORT_DRAFTED=blue, AWAITING_APPROVAL=amber, APPROVED=green, REJECTED=orange, FAILED=red.

Each phase panel renders only when its data is present on the row record. A study in `RUNNING_CONTINGENCY` shows panel 1 but not yet panel 2 (panel 2 appears with a spinner until the contingency task returns). This is the visual proof that the typed handoff between phases is the only path information travels.
