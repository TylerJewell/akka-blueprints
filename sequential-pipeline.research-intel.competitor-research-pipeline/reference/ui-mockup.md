# UI mockup — competitor-research-pipeline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Competitor Research Pipeline</title>`.

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
- Headline: `Competitor Research <span class="accent">Pipeline</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded competitors (or type your own) and click **Run pipeline**.
  3. Watch the card transition through SEARCHING → SEARCHED → SUMMARIZING → SUMMARIZED → PUBLISHING → PUBLISHED → EVALUATED.
  4. Open the card detail to see the Notion page URL and the eval score; check the rejection-log strip if any guardrail violations fired.
- Card **How it works**: one paragraph on the three task phases (SEARCH → SUMMARIZE → PUBLISH) and the typed handoff between them; one paragraph on the guardrail (phase-gate + Notion schema/scope check) and the completeness evaluator.
- Card **Components**: rows per component (CompetitorEntity, CompetitorPipelineWorkflow, ResearchAgent, SearchTools, SummarizeTools, PublishTools, NotionWriteGuardrail, PublishQualityScorer, CompetitorView, CompetitorEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (competitor names are public entities; no personal data is processed). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. The Exa.ai and Notion service fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Name a competitor. <span class="accent">Get the profile.</span>`. Subtitle: `One agent, three task phases, one runtime gate before the Notion write.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Competitor name` (with a "Pick a seeded competitor" dropdown that fills it), and a yellow `Run pipeline` button.
    - Live list below: one card per competitor, newest-first. Each card shows status pill, eval score chip (when eval landed), competitor name, age, and a small red dot if any guardrail rejection fired.
  - **Right column** — Selected-competitor detail.
    - Header: status pill + eval score chip + competitor name.
    - Phase panel 1 (Search results): a table with columns title, url, excerpt. Visible once `searchResults.isPresent()`.
    - Phase panel 2 (Profile summary): a field list with five rows (pricing model, primary use case, notable integrations, known differentiators, data residency stance) each showing value and supporting URL chip. Visible once `summary.isPresent()`.
    - Phase panel 3 (Competitor profile): one-liner, the five field rows, and a Notion page URL link. Visible once `profile.isPresent()`.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the competitor has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, SEARCHING=blue, SEARCHED=blue, SUMMARIZING=yellow, SUMMARIZED=yellow, PUBLISHING=blue, PUBLISHED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A competitor in `SUMMARIZING` shows panels 1 and 2 (panel 2 with a "in progress" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
