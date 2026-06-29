# UI mockup — long-rag-workflow

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Long RAG Workflow</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Long RAG <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded queries (or type your own) and click **Run pipeline**.
  3. Watch the card transition through RETRIEVING → RETRIEVED → SYNTHESIZING → SYNTHESIZED → REPORTING → REPORTED → EVALUATED.
  4. Inspect the citation-flag log strip on the card if any uncited passages were flagged during generation.
- Card **How it works**: one paragraph on the three task phases (RETRIEVE → SYNTHESIZE → REPORT) and the typed handoff between them for long documents; one paragraph on the two governance mechanisms (citation guardrail, coverage eval).
- Card **Components**: rows per component (QueryEntity, LongRagPipelineWorkflow, DocumentAgent, RetrieveTools, SynthesizeTools, ReportTools, CitationGuardrail, CoverageScorer, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the query text is the only user input; document corpus comes from an in-process store). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, E1). ID badges coloured: H1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">Read the report.</span>`. Subtitle: `One agent, three task phases, citation grounding enforced on every response.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Research query` (with a "Pick a seeded query" dropdown that fills it), and a yellow `Run pipeline` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, coverage score chip (when eval landed), query text (truncated to 60 chars), age, and a small orange dot if any citation flags fired during generation.
  - **Right column** — Selected-query detail.
    - Header: status pill + coverage score chip + query text.
    - Phase panel 1 (Retrieved chunks): a table with columns doc title, page range, chunk id, text preview (first 80 chars). Visible once `chunkWindow` is present.
    - Phase panel 2 (Synthesis): a themes list and a findings list with theme assignments and supporting chunk ids. Visible once `synthesis` is present.
    - Phase panel 3 (Report): title, abstract paragraph, then a per-section block (heading, body with inline `[chunkId]` tags highlighted, citation chips). Visible once `report` is present.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Citation-flag log strip (only visible if the query has any `citationFlags`): a small table with flagged sentence (truncated), missing citation note, window size, and timestamp.
- Status pill colours: CREATED=muted, RETRIEVING=blue, RETRIEVED=blue, SYNTHESIZING=yellow, SYNTHESIZED=yellow, REPORTING=blue, REPORTED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A query in `SYNTHESIZING` shows panels 1 and 2 (panel 2 with an "in progress" spinner if the agent has not yet returned). The inline `[chunkId]` tag highlights in the report body are the visual proof that the citation guardrail was in the loop — every tagged passage traces back to a row in the chunks table.
