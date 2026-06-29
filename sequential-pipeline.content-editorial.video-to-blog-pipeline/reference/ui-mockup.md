# UI mockup — video-to-blog-pipeline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: YouTube to Blog</title>`.

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
- Headline: `YouTube to <span class="accent">Blog</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded video URLs (or paste your own) and click **Convert to blog post**.
  3. Watch the card transition through TRANSCRIBING → TRANSCRIBED → SUMMARISING → SUMMARISED → DRAFTING → DRAFTED → POLISHING → POLISHED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any publish-guardrail rejections fired.
- Card **How it works**: one paragraph on the four task phases (TRANSCRIPT → SUMMARISE → DRAFT → POLISH) and the typed handoff between them; one paragraph on the two governance mechanisms (publish-content guardrail, editorial quality eval).
- Card **Components**: rows per component (BlogPostEntity, BlogPipelineWorkflow, BlogAgent, TranscriptTools, SummaryTools, DraftTools, PolishTools, PublishGuardrail, EditorialScorer, BlogPostView, BlogEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 4 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the video URL and transcript do not contain personal data in the typical editorial use case). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Paste a URL. <span class="accent">Read the post.</span>`. Subtitle: `One agent, four task phases, one content gate before the post is recorded.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `YouTube URL` (with a "Pick a seeded URL" dropdown that fills it), and a yellow `Convert to blog post` button.
    - Live list below: one card per post, newest-first. Each card shows status pill, eval score chip (when eval landed), video URL (truncated), age, and a small red dot if any guardrail rejection fired during this post.
  - **Right column** — Selected-post detail.
    - Header: status pill + eval score chip + video URL.
    - Phase panel 1 (Transcript): text preview (first 300 chars), duration, chapter list. Visible once `transcript` is present.
    - Phase panel 2 (Summary): key-point list with source-timestamp badges, section outline. Visible once `summary` is present.
    - Phase panel 3 (Draft): introduction text, per-section headings and raw bodies. Visible once `draft` is present.
    - Phase panel 4 (Published post): title, polished introduction, per-section blocks (heading, body, source-timestamp chips), conclusion. Visible once `post` is present.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the post has any `guardrailRejections`): a small table with phase, reason, time.
- Status pill colours: CREATED=muted, TRANSCRIBING=blue, TRANSCRIBED=blue, SUMMARISING=yellow, SUMMARISED=yellow, DRAFTING=blue, DRAFTED=blue, POLISHING=yellow, POLISHED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A post in `DRAFTING` shows panels 1, 2, and 3 (panel 3 with a spinner if the draft is still in progress). This is the visual proof that the typed handoff between phases is the only path information travels.
