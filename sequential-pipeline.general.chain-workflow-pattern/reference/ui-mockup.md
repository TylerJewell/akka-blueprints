# UI mockup — chain-workflow

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Chain Workflow</title>`.

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

And the DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Chain <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded inputs (or paste your own) and click **Run chain**.
  3. Watch the card transition through EXTRACTING → EXTRACTED → REFINING → REFINED → FORMATTING → FORMATTED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any output-validation rejections fired.
- Card **How it works**: one paragraph on the three task stages (EXTRACT → REFINE → FORMAT) and the typed handoff between them; one paragraph on the two governance mechanisms (output-validation guardrail, quality eval).
- Card **Components**: rows per component (DocumentEntity, TransformPipelineWorkflow, TransformAgent, ExtractTools, RefineTools, FormatTools, OutputGuardrail, QualityScorer, DocumentView, DocumentEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the raw input is user-supplied text, not person-level). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Paste content. <span class="accent">Get a document.</span>`. Subtitle: `One agent, three transformation stages, one runtime validator between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: textarea `Raw input` (with a "Pick a seeded input" dropdown that fills it), and a yellow `Run chain` button.
    - Live list below: one card per document, newest-first. Each card shows status pill, quality score chip (when eval landed), input label (first 40 chars), age, and a small red dot if any guardrail rejection fired during this document.
  - **Right column** — Selected-document detail.
    - Header: status pill + quality score chip + input label.
    - Stage panel 1 (Extracted facts): a table with columns factId, text, category, sourceSpan. Visible once `extracted.isPresent()`.
    - Stage panel 2 (Refined content): a refined-fact list with refinedId, text, sourceFactId, polishNote. Visible once `refined.isPresent()`.
    - Stage panel 3 (Document): title, summary paragraph, then a per-section block (heading, body, citations chips). Visible once `document.isPresent()`.
    - Quality section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the document has any `guardrailRejections`): a small table with stage, failedField, reason, time.
- Status pill colours: CREATED=muted, EXTRACTING=blue, EXTRACTED=blue, REFINING=yellow, REFINED=yellow, FORMATTING=blue, FORMATTED=blue, EVALUATED=green, FAILED=red.

Each stage panel renders only when its data is present on the row record. A document in `REFINING` shows panels 1 and 2 (panel 2 with a "in progress" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between stages is the only path information travels.
