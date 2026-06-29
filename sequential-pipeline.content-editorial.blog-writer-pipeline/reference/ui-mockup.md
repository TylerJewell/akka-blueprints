# UI mockup — blog-writer-pipeline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Blog Writer Pipeline</title>`.

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
- Headline: `Blog Writer <span class="accent">Pipeline</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded topics (or type your own), choose a style, and click **Generate post**.
  3. Watch the card transition through RESEARCHING → RESEARCHED → OUTLINING → OUTLINED → DRAFTING → DRAFTED → QUALITY_CHECKED.
  4. Inspect the rejection-log strip on the card if any brand-voice rejections fired during the draft phase.
- Card **How it works**: one paragraph on the three task phases (RESEARCH → OUTLINE → DRAFT) and the typed handoff between them; one paragraph on the two governance mechanisms (brand-voice guardrail, quality eval).
- Card **Components**: rows per component (BlogPostEntity, BlogWritingWorkflow, BlogWriterAgent, ResearchTools, OutlineTools, DraftTools, BrandGuardrail, QualityScorer, BlogPostView, BlogPostEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the topic is the only user input; references come from an in-process corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Type a topic. <span class="accent">Read the post.</span>`. Subtitle: `One agent, three task phases, one brand guardrail between draft and record.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Topic` (with a "Pick a seeded topic" dropdown that fills it), a `Style` dropdown (`Technical` / `Conversational` / `Thought leadership`), and a yellow `Generate post` button.
    - Live list below: one card per post, newest-first. Each card shows status pill, quality score chip (when quality landed), topic, style badge, age, and a small red dot if any brand guardrail rejection fired during this post.
  - **Right column** — Selected-post detail.
    - Header: status pill + quality score chip + topic + style badge.
    - Phase panel 1 (Research): a table with columns source, url, keyInsight. Visible once `research` is present.
    - Phase panel 2 (Outline): a sections list showing each section's heading and key points. Visible once `outline` is present.
    - Phase panel 3 (Draft): title, introduction paragraph, then a per-section block (heading, body text), then conclusion paragraph. Visible once `post` is present.
    - Quality section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red and shows a "needs editor review" badge.
    - Rejection-log strip (only visible if the post has any `guardrailRejections`): a small table with rule, detail, time.
- Status pill colours: CREATED=muted, RESEARCHING=blue, RESEARCHED=blue, OUTLINING=yellow, OUTLINED=yellow, DRAFTING=blue, DRAFTED=blue, QUALITY_CHECKED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A post in `OUTLINING` shows panels 1 and 2 (panel 2 with a spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
