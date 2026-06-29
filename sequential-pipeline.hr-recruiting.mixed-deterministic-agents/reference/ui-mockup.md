# UI mockup — score-aggregator

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Non-AI Agents (Score Aggregator)</title>`.

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
- Headline: `Non-AI Agents <span class="accent">(Score Aggregator)</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded candidates (or enter your own) and select a target role, then click **Submit application**.
  3. Watch the card transition through SCREENING → SCREENED → SCORING → SCORED → RECOMMENDING → RECOMMENDED → NOTIFIED.
  4. Check the gate result chip — green PASS means all four QualityGate checks passed; red FAIL names the first failing check.
- Card **How it works**: one paragraph on the four pipeline phases (SCREEN → SCORE → RECOMMEND → NOTIFY) and where the LLM is used vs. deterministic Java; one paragraph on the CI test gate and the runtime QualityGate.
- Card **Components**: rows per component (ApplicationEntity, ApplicationPipelineWorkflow, ScreeningAgent, ScoreTools, RecommendTools, ScoreAggregator, StatusNotifier, QualityGate, ApplicationView, ApplicationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type. Deterministic Java classes (ScoreAggregator, StatusNotifier, QualityGate) are visually distinguished from the Akka primitives.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 2 function-tool classes, 2 deterministic Java agents, and 1 QualityGate as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled (candidateName and resumeText are personal data). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic. The `deterministic_components` list under Operations is highlighted to make the mixed-agent property visible.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (H1). ID badge coloured green (CI gate / build-gate).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a candidate. <span class="accent">See the pipeline.</span>`. Subtitle: `One LLM agent, two deterministic Java agents, one build gate between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Candidate name`, textarea `Resume text`, dropdown `Target role` (seeded: Staff Software Engineer, Product Manager, Data Scientist), and a yellow `Submit application` button.
    - "Pick a seeded candidate" shortcut that fills name, resume, and role at once.
    - Live list below: one card per application, newest-first. Each card shows status pill, gate result chip (when gate landed), candidate name, role, and age.
  - **Right column** — Selected-application detail.
    - Header: status pill + gate result chip + candidate name + role.
    - Pipeline-phase timeline strip: four phase badges (SCREEN — LLM, SCORE — Java, RECOMMEND — LLM, NOTIFY — Java) coloured by component type to make the mixed-agent property visible at a glance.
    - Phase panel 1 (Screen result): skills match table (skill, matched checkmark, evidence snippet) + fit summary. Visible once `screenResult` is present.
    - Phase panel 2 (Candidate score): dimension score table (dimension, score, max, rationale) + total score bar. Visible once `candidateScore` is present.
    - Phase panel 3 (Recommendation): decision badge (ADVANCE/HOLD/REJECT with appropriate colour) + rationale sentence. Visible once `recommendation` is present.
    - Phase panel 4 (Status update): notification payload display + `notifiedAt` timestamp. Visible once `statusUpdate` is present.
    - Gate result section at bottom: PASS/FAIL badge + reason sentence. FAIL highlights the card border orange.
- Status pill colours: CREATED=muted, SCREENING=blue, SCREENED=blue, SCORING=yellow, SCORED=yellow, RECOMMENDING=blue, RECOMMENDED=blue, NOTIFIED=green, FAILED=red.
- Gate result chip colours: PASS=green, FAIL=orange.

Each phase panel renders only when its data is present on the row record. A briefing in `RECOMMENDING` shows panels 1 and 2 but not 3 or 4. The pipeline-phase timeline strip updates in real time — each badge lights up when its phase completes.
