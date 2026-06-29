# UI mockup — wp-autotagger

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: WP Autotagger</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `WP Auto<span class="accent">Tagger</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded post example (tech / product-review / news) and load it with one click.
  3. Click **Submit for tagging**.
  4. Watch the card transition through BODY_FETCHED → TAGGING → TAGS_APPLIED.
- Card **How it works**: one paragraph on submit → fetch → tag → apply; one paragraph on the governance mechanism (before-tool-call guardrail, what it checks, what a rejection triggers).
- Card **Components**: rows per component (PostEntity, WpApiConsumer, WpApiStub, TaggingWorkflow, TagProposerAgent, TagProposalGuardrail, PostView, PostEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Stub as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `automated` authority level and `human_in_loop = false` declarations are filled and prominent. `credential_tokens_redacted_before_llm = true` is highlighted in the Data section. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a post. <span class="accent">Get SEO tags.</span>`. Subtitle: `One agent, one guardrail at the write boundary.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: **Post URL** text input, **Max tags** number input (1–15, default 10), **Tag style** dropdown (DESCRIPTIVE / KEYWORD_DENSE / MIXED), **Submitted by** text input, and a "Load seeded example" set of three quick-fill links (tech / product-review / news). Yellow **Submit for tagging** button.
    - Live list below: one card per tagging job, newest-first. Each card shows status pill, tag count badge (when tags are applied), post title (from fetched body), age.
  - **Right column** — Selected-job detail.
    - Header: status pill + tag count badge + post title.
    - Post metadata: URL, author, published date.
    - Post body preview: monospace block of first 500 chars of the fetched body, with a note if credential tokens were redacted.
    - Tagging config: maxTags, tagStyle, submittedBy.
    - Tag chips: one chip per `TagEntry`, ordered by confidence. Each chip shows the tag text and a small confidence bar (0–100%). Chips are coloured by confidence: ≥ 0.9 = green, 0.7–0.89 = blue, < 0.7 = muted.
    - Rationale: the agent's 1–2-sentence paragraph.
- Status pill colours: INGESTED=muted, BODY_FETCHED=blue, TAGGING=yellow, TAGS_APPLIED=green, FAILED=red.
- Tag count badge: grey background, white text, e.g. `5 tags`.

The raw post body (including any redacted tokens) is not displayed beyond the 500-char preview. Editors who need the full body fetch `/api/posts/{id}` and read `body.body` from the JSON.
