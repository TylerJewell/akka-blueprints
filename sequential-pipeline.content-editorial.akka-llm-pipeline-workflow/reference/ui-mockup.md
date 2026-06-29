# UI mockup — akka-llm-pipeline-workflow

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Deterministic Workflow with LLM Activities</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Deterministic Workflow with <span class="accent">LLM Activities</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded topics (or type your own) and set an optional word target, then click **Generate post**.
  3. Watch the card transition through OUTLINING → OUTLINED → DRAFTING → DRAFTED → REVIEWING → READY (or FLAGGED).
  4. Inspect the editorial-review chip on the card to see whether the draft passed or was flagged.
- Card **How it works**: one paragraph on the two activity phases (OUTLINE → DRAFT) and the typed handoff between them; one paragraph on the two governance mechanisms (schema guardrail on the outline, editorial guardrail on the draft).
- Card **Components**: rows per component (PostEntity, ContentWorkflow, OutlineActivity, DraftActivity, SchemaGuardrail, EditorialGuardrail, PostView, PostEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (0 AutonomousAgents, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 2 LLM activities, 2 Guardrails as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (only the topic string and word target are user inputs). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. `operations.agent_count = 0` highlights that there are no autonomous agents. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, H2). ID badges coloured: H1 red (guardrail · after-llm-response), H2 red (guardrail · before-agent-response).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Type a topic. <span class="accent">Read the post.</span>`. Subtitle: `Two LLM activities, two guardrails, one typed handoff between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Topic` (with a "Pick a seeded topic" dropdown that fills it), a number input `Word target` (default 600), and a yellow `Generate post` button.
    - Live list below: one card per post, newest-first. Each card shows status pill, editorial-review chip (PASSED / FLAGGED when review landed), topic, age.
  - **Right column** — Selected-post detail.
    - Header: status pill + editorial-review chip + topic.
    - Phase panel 1 (Outline): a sections table with columns heading and key-points. Visible once `outline` is present on the row.
    - Phase panel 2 (Draft): title, introduction paragraph, then per-section blocks (heading, body). Visible once `draft` is present.
    - Editorial review section: a PASSED / FLAGGED badge and the one-line reason. Visible once `review` is present; FLAGGED highlights the card border in amber.
    - Validation-failure strip (only visible if `validationFailure` is non-null): field name, reason, time. Replaces phase panels 1 and 2 when the post is in `FAILED` state due to schema rejection.
- Status pill colours: CREATED=muted, OUTLINING=blue, OUTLINED=blue, DRAFTING=yellow, DRAFTED=yellow, REVIEWING=blue, READY=green, FLAGGED=amber, FAILED=red.

Each phase panel renders only when its data is present on the row record. A post in `DRAFTING` shows phase panel 1 (outline) and a spinner for phase panel 2. This is the visual proof that the typed handoff between activities is the only path information travels — the draft panel is empty until the draft activity returns.
