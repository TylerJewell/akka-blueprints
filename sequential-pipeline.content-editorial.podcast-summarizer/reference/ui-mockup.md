# UI mockup — podcast-summarizer

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Podcast Transcript Summarizer</title>`.

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
- Headline: `Podcast Transcript <span class="accent">Summarizer</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded episodes (or paste your own transcript) and click **Run pipeline**.
  3. Watch the card transition through EXTRACTING → EXTRACTED → SEGMENTING → SEGMENTED → SUMMARIZING → SUMMARIZED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any phase-gate rejections fired.
- Card **How it works**: one paragraph on the three task phases (EXTRACT → SEGMENT → SUMMARIZE) and the typed handoff between them; one paragraph on the two governance mechanisms (phase-gate guardrail, fidelity eval).
- Card **Components**: rows per component (EpisodeEntity, SummaryPipelineWorkflow, TranscriptAgent, ExtractTools, SegmentTools, SummarizeTools, PhaseGuardrail, FidelityScorer, EpisodeView, EpisodeEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (quotes are public podcast content; no personal data is processed). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers — an editor reads every summary before publication. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Paste a transcript. <span class="accent">Read the summary.</span>`. Subtitle: `One agent, three task phases, one runtime gate between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Episode title` plus a `Transcript` textarea (with a "Pick a seeded episode" dropdown that fills both), and a yellow `Run pipeline` button.
    - Live list below: one card per episode, newest-first. Each card shows status pill, eval score chip (when eval landed), episode title, age, and a small red dot if any guardrail rejection fired during this episode.
  - **Right column** — Selected-episode detail.
    - Header: status pill + eval score chip + episode title.
    - Phase panel 1 (Extracted quotes): a table with columns speaker, timestamp, text. Visible once `quotes.isPresent()`.
    - Phase panel 2 (Topic segments): a cluster list showing each cluster's label and the count of assigned quotes. Visible once `segmentation.isPresent()`.
    - Phase panel 3 (Summary): overall takeaway paragraph, then a per-section block (heading, body, supporting quote ids as chips). Visible once `summary.isPresent()`.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the episode has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, EXTRACTING=blue, EXTRACTED=blue, SEGMENTING=yellow, SEGMENTED=yellow, SUMMARIZING=blue, SUMMARIZED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. An episode in `SEGMENTING` shows panels 1 and 2 (panel 2 with a spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
