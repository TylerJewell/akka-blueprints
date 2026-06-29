# UI mockup — product-seo-enricher

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Product SEO Enricher</title>`.

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
- Headline: `Product SEO <span class="accent">Enricher</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded products (or type your own) and click **Run enrichment**.
  3. Watch the card transition through FETCHING → FETCHED → ANALYZING → ANALYZED → ENRICHING → ENRICHED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any phase-gate rejections fired.
- Card **How it works**: one paragraph on the three task phases (FETCH → ANALYZE → ENRICH) and the typed handoff between them; one paragraph on the browser-tool phase-gate guardrail and the deterministic quality evaluator.
- Card **Components**: rows per component (EnrichmentEntity, EnrichmentPipelineWorkflow, SeoAgent, FetchTools, AnalyzeTools, EnrichTools, BrowserGuardrail, QualityScorer, EnrichmentView, EnrichmentEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the product name is the only user input; SERP data comes from an in-process fixture corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a product. <span class="accent">Get SEO recommendations.</span>`. Subtitle: `One agent, three task phases, one runtime gate between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Product name` (with a "Pick a seeded product" dropdown that fills it), and a yellow `Run enrichment` button.
    - Live list below: one card per enrichment, newest-first. Each card shows status pill, quality score chip (when eval landed), product name, age, and a small red dot if any guardrail rejection fired during this enrichment.
  - **Right column** — Selected-enrichment detail.
    - Header: status pill + quality score chip + product name.
    - Phase panel 1 (SERP entries): a table with columns position, title, url, snippet. Visible once `serpResult.isPresent()`.
    - Phase panel 2 (SERP Analysis): a keyword candidates list (keyword, frequency, sourceUrl) and a competitor signals list (domain, position, angle). Visible once `serpAnalysis.isPresent()`.
    - Phase panel 3 (Enrichment): ranked keywords table (keyword, relevance score, rationale), competitor summary paragraph, recommended meta description with character-count badge. Visible once `enrichment.isPresent()`.
    - Quality section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the enrichment has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, FETCHING=blue, FETCHED=blue, ANALYZING=yellow, ANALYZED=yellow, ENRICHING=blue, ENRICHED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. An enrichment in `ANALYZING` shows phase panels 1 and 2 (panel 2 with a spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
