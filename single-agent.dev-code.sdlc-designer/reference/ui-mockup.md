# UI mockup — sdlc-technical-designer

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SDLC Technical Designer</title>`.

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
- Headline: `SDLC Technical <span class="accent">Designer</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded project context (microservices / monolith / event-driven) and a seeded feature description, or load both with one click.
  3. Click **Submit for design**.
  4. Watch the card transition through CONTEXT_LOADED → DESIGNING → PROPOSAL_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → load context → design → eval; one paragraph on the two governance mechanisms (before-agent-response guardrail, on-decision eval).
- Card **Components**: rows per component (DesignRequestEntity, ContextLoader, DesignWorkflow, DesignAgent, ProposalGuardrail, ProposalScorer, DesignView, DesignEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` declarations are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic. The `pii: false` declaration in Data is prominent since this blueprint processes feature descriptions, not personal data.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a feature. <span class="accent">Get a design.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Project context` (microservices / monolith / event-driven / custom), `Feature title` text input, `Feature description` textarea (with a "Load seeded example" link that fills both title and body), optional `Constraints` textarea (JSON array of TechConstraint objects), `Requested by` text input, and a yellow `Submit for design` button.
    - Live list below: one card per design request, newest-first. Each card shows status pill, eval score chip (when eval landed), feature title, age.
  - **Right column** — Selected-request detail.
    - Header: status pill + eval score chip + `featureTitle`.
    - Project context strip: architecture pattern badge, existing components list, preferred patterns chips, target language chip.
    - Submitted constraints: a small table with `constraintId`, `category` chip, and description.
    - Proposal — Component table: columns componentId, componentKind (coloured chip), dependsOn, rationale.
    - Proposal — Data model: one collapsible sub-section per `DataModelSketch` entity, showing fields table (name, type, required badge, purpose) and event types chips.
    - Proposal — API surface: a table of method, path, requestSummary, responseSummary, owningComponent.
    - Decision log: an expandable list; each entry shows decisionId, componentId, decision text, rationale, and alternatives-considered chips.
    - Executive summary: a paragraph at the top of the proposal section.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, CONTEXT_LOADED=blue, DESIGNING=yellow, PROPOSAL_RECORDED=blue, EVALUATED=green, FAILED=red.
- ComponentKind chip colours: EventSourcedEntity=gold, Workflow=purple, HttpEndpoint=white, View=green, Consumer=orange, AutonomousAgent=cyan, TimedAction=grey.
- Eval score chip: 1–2=red, 3=yellow, 4–5=green.
