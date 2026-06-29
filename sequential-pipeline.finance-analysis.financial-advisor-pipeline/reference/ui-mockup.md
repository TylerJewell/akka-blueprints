# UI mockup — financial-advisor-pipeline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Financial Advisor Pipeline</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Financial Advisor <span class="accent">Pipeline</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded queries (or type your own) and click **Run pipeline**.
  3. Watch the card transition through RESEARCHING → RESEARCHED → STRATEGIZING → STRATEGIZED → PLANNING → PLANNED → ASSESSING → EVALUATED.
  4. Inspect the disclaimer-log strip and the sanitizer-event strip on the selected advisory's right pane.
- Card **How it works**: one paragraph on the four task phases (RESEARCH → STRATEGY → EXECUTE → ASSESS) and the typed handoff between them; one paragraph on the two governance mechanisms (disclaimer guardrail on every response, sector sanitizer that redacts prohibited financial-sector language).
- Card **Components**: rows per component (AdvisoryEntity, AdvisorPipelineWorkflow, FinancialAdvisorAgent, ResearchTools, StrategyTools, ExecutionTools, RiskTools, DisclaimerGuardrail, SectorSanitizer, ComplianceScorer, AdvisoryView, AdvisoryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 4 function-tool classes, 1 Guardrail, 1 Sanitizer, 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `sector: finance` declaration is filled. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive pre-filled answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are rendered in muted italic — deployers who adapt this blueprint must supply jurisdiction, retention, and licensing answers.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 purple (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the analysis.</span>`. Subtitle: `One agent, four task phases, two runtime governance layers on every response.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Financial query` (with a "Pick a seeded query" dropdown that fills it), and a yellow `Run pipeline` button.
    - Live list below: one card per advisory, newest-first. Each card shows status pill, compliance score chip (when scored), query excerpt, age, and a small purple dot if any sanitizer event fired during this advisory.
  - **Right column** — Selected-advisory detail.
    - Header: status pill + compliance score chip + query excerpt.
    - Phase panel 1 (Market Research): a table with columns sector, metric, value, source. Visible once `snapshot` is present.
    - Phase panel 2 (Strategy): allocation targets table (asset class, target%, rationale) and rationale items list. Visible once `strategy` is present.
    - Phase panel 3 (Execution Plan): ordered action-item list (sequence, description, timing, instruments). Visible once `executionPlan` is present.
    - Phase panel 4 (Risk Assessment): risk band badge, volatility metrics (stdDev, beta, Sharpe), and mitigations list. Visible once `riskProfile` is present.
    - Compliance section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Disclaimer-log strip: a small table with injectedAt timestamp and first 80 chars of disclaimer text. Always visible once any disclaimer has been injected (non-zero `disclaimerLog`).
    - Sanitizer-event strip (only visible if `sanitizerLog` is non-empty): a small table with firedAt, matchedPattern, and redactedFragment.
- Status pill colours: CREATED=muted, RESEARCHING=blue, RESEARCHED=blue, STRATEGIZING=yellow, STRATEGIZED=yellow, PLANNING=blue, PLANNED=blue, ASSESSING=yellow, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. An advisory in `PLANNING` shows panels 1 and 2 (panel 3 with a "in progress" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels between task boundaries.
