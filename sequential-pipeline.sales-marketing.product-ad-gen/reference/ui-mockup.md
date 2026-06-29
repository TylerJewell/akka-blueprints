# UI mockup — product-catalog-ad-generation

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Product Catalog Ad Generation</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Product Catalog <span class="accent">Ad Generation</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded products (or type your own product name) and click **Generate Ad**.
  3. Watch the card transition through ENRICHING → ENRICHED → DRAFTING → DRAFTED → REVIEWING → APPROVED → SCORED.
  4. Inspect the rejection-log strip on the card if any brand-policy rejections fired during drafting.
- Card **How it works**: one paragraph on the three task phases (ENRICH → DRAFT → REVIEW) and the typed handoff between them; one paragraph on the brand-policy guardrail and the deterministic compliance scorer.
- Card **Components**: rows per component (AdJobEntity, AdGenerationWorkflow, AdGeneratorAgent, EnrichTools, DraftTools, ReviewTools, BrandGuardrail, BrandComplianceScorer, AdJobView, AdJobEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the product name is the only user input; catalog data is in-process). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Deployer-specific fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Pick a product. <span class="accent">Generate the ad.</span>`. Subtitle: `One agent, three task phases, one brand gate before the draft lands.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Product name` (with a "Pick a seeded product" dropdown that fills it), and a yellow `Generate Ad` button.
    - Live list below: one card per job, newest-first. Each card shows status pill, compliance score chip (when scored), product name, age, and a small red dot if any guardrail rejection fired during this job.
  - **Right column** — Selected-job detail.
    - Header: status pill + compliance score chip + product name.
    - Phase panel 1 (Enriched product): attribute table with columns category, pricingTier, keyFeatures, targetAudience. Visible once `enrichedProduct` is present.
    - Phase panel 2 (Ad draft): headline, callToAction, and a placement card per entry (type badge, copy text, char count vs limit bar). Visible once `adDraft` is present.
    - Phase panel 3 (Final ad): headline, approved copy per placement. Visible once `productAd` is present.
    - Compliance section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the job has any `guardrailRejections`): a small table with phase, field, rule, found, and time.
- Status pill colours: CREATED=muted, ENRICHING=blue, ENRICHED=blue, DRAFTING=yellow, DRAFTED=yellow, REVIEWING=blue, APPROVED=blue, SCORED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A job in `DRAFTING` shows panel 1 (complete) and panel 2 (spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
