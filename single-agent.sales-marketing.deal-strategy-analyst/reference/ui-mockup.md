# UI mockup — deal-strategy-analyst

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: DealStrategyAnalyst</title>`.

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
- Headline: `Deal<span class="accent">Strategy</span>Analyst`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded deal type (enterprise-new / renewal-risk / competitive) and load the matching seed context with one click.
  3. Click **Analyse deal**.
  4. Watch the card transition through SANITIZED → ANALYSING → RECOMMENDATION_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → sanitize → analyse → eval; one paragraph on the two governance mechanisms (sanitizer, guardrail) and the on-decision eval.
- Card **Components**: rows per component (DealEntity, DealContextSanitizer, AnalysisWorkflow, DealStrategyAgent, RecommendationGuardrail, RecommendationScorer, DealView, DealEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` and `nda-restricted-company-names: true` declarations in Data are filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and displayed in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, E1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a deal. <span class="accent">Get a strategy.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Deal type` (enterprise-new / renewal-risk / competitive / custom), `Deal name` text input, `Context` textarea (with a "Load seeded example" link that fills both the deal name and the context body), `Stakeholders` section (add/remove rows with role, concern, engagementLevel inputs), `Submitted by` text input, and a yellow `Analyse deal` button.
    - Live list below: one card per deal, newest-first. Each card shows status pill, urgency badge (when recommendation landed), eval score chip (when eval landed), deal name, age.
  - **Right column** — Selected-deal detail.
    - Header: status pill + urgency badge + eval score chip + `dealName`.
    - Submitted stakeholders: a compact list with role, engagementLevel chip, and concern text.
    - Sanitized context preview: a monospace block of the redacted context, with PII category chips above (`email`, `phone`, `person-name`, etc.).
    - Recommendation summary: the agent's executive-summary paragraph.
    - Next steps table: columns priority, action type (coloured chip), stakeholder role, rationale, suggested deadline. Rows sorted by priority ascending.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, ANALYSING=yellow, RECOMMENDATION_RECORDED=blue, EVALUATED=green, FAILED=red.
- Urgency badge colours: CRITICAL=red, HIGH=yellow, NORMAL=muted.
- Action type chip colours: ESCALATE=red, ADDRESS_OBJECTION=yellow, SCHEDULE_CALL=blue, all others=muted.

The raw deal context is never displayed on this screen — only the sanitized form. Reps who need the raw text can fetch `/api/deals/{id}` and read `context.rawContext` from the JSON.
