# UI mockup — AI Blog Writer Pipeline with Ollama

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: AI Blog Writer Pipeline with Ollama</title>`.

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
- Headline: `AI Blog Writer <span class="accent">Pipeline</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded topics (or type your own), choose a post type, and click **Run pipeline**.
  3. Watch the card transition through RESEARCHING → OUTLINED → DRAFTED → EDITED → PUBLISHED.
  4. Inspect the block-log strip on the card if any content-policy blocks fired.
- Card **How it works**: one paragraph on the five task phases (RESEARCH → OUTLINE → DRAFT → EDIT → PUBLISH) and the typed handoff between them; one paragraph on the content-policy guardrail that checks each phase's prose before it is written to the entity.
- Card **Components**: rows per component (PostEntity, BlogPipelineWorkflow, BlogWriterAgent, ResearchTools, OutlineTools, DraftTools, EditTools, PublishTools, ContentPolicyGuardrail, PostView, PostEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 5 function-tool classes, 1 Guardrail as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the topic is the only user input; references come from an in-process corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Type a topic. <span class="accent">Read the post.</span>`. Subtitle: `One agent, five task phases, one policy gate on every response.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Topic` (with a "Pick a seeded topic" dropdown that fills it), a `Post type` dropdown (`tutorial` / `explainer` / `opinion`), and a yellow `Run pipeline` button.
    - Live list below: one card per post, newest-first. Each card shows status pill, post type badge, topic, age, and a small red dot if any guardrail block fired during this post.
  - **Right column** — Selected-post detail.
    - Header: status pill + post type badge + topic.
    - Phase panel 1 (Research notes): a table with columns source, url, summary. Visible once `research.isPresent()`.
    - Phase panel 2 (Outline): a sections list with heading and key points. Visible once `outline.isPresent()`.
    - Phase panel 3 (Draft): title, introduction, then per-section prose. Visible once `draft.isPresent()`.
    - Phase panel 4 (Edited draft): title, introduction, then per-section prose with readability score chips. Visible once `editedDraft.isPresent()`.
    - Phase panel 5 (Published post): title, summary, then per-section block (heading, body, reference chips). Visible once `post.isPresent()`.
    - Policy check section at bottom: a green/red pill showing the last policy-check status and a one-line reason. A `FAILED` policy check highlights the card border red.
    - Block-log strip (only visible if the post has any `guardrailBlocks`): a small table with phase, reason, time.
- Status pill colours: CREATED=muted, RESEARCHING=blue, RESEARCHED=blue, OUTLINING=yellow, OUTLINED=yellow, DRAFTING=yellow, DRAFTED=yellow, EDITING=blue, EDITED=blue, PUBLISHING=blue, PUBLISHED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A post in `OUTLINING` shows panels 1 and 2 (panel 2 with a spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
