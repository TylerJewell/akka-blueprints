# UI mockup — spec-to-pr

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Spec to Implementation PR</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Spec to Implementation <span class="accent">PR</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded specs (or paste your own) and click **Run pipeline**.
  3. Watch the card transition through PARSING → PLANNED → DRAFTING → AWAITING_REVIEW.
  4. Click **Approve** in the right pane to advance through CI to MERGE_READY (or **Reject** to terminate).
- Card **How it works**: one paragraph on the three task phases (PARSE → PLAN → DRAFT) and the typed handoff between them; one paragraph on the three governance mechanisms (phase-gate guardrail, human review hold, CI gate).
- Card **Components**: rows per component (SpecRunEntity, SpecPipelineWorkflow, ImplementationAgent, ParseTools, PlanTools, DraftTools, WriteGuardrail, CiScorer, SpecRunView, SpecEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (specs are code-level, not person-level). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. `oversight.reviewer_must_approve_before_ci = true` is highlighted. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, C1). ID badges coloured: G1 red (guardrail), H1 indigo (HITL), C1 orange (CI gate).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Paste a spec. <span class="accent">Get a PR.</span>`. Subtitle: `One agent, three task phases, a human review hold, and a CI gate.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: textarea `Spec text` (with a "Pick a seeded spec" dropdown that fills it), and a yellow `Run pipeline` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, spec title (first 60 chars), age, and a small red dot if any guardrail rejection fired. Cards in `AWAITING_REVIEW` show a blue "review needed" indicator.
  - **Right column** — Selected-run detail.
    - Header: status pill + spec title.
    - Phase panel 1 (Parsed spec): a requirements table with columns reqId, text, priority, affectedFiles. Visible once `parsedSpec.isPresent()`.
    - Phase panel 2 (Change plan): a file-change table with columns filePath, changeType, rationale. Visible once `changePlan.isPresent()`.
    - Phase panel 3 (Draft PR): title, description block, per-file diff block (filePath chip + proposedDiff in a `<pre>`). Visible once `draftPr.isPresent()`.
    - Reviewer action bar (only active in `AWAITING_REVIEW`): reviewer ID input, comment textarea, green **Approve** button, red **Reject** button. After a decision is submitted, the bar is replaced by the review outcome badge (approved / rejected) with the reviewer ID and comment.
    - CI panel: a 3-row check table (compile / test / lint) with pass/fail badges and message text. Visible once `ciResult.isPresent()`. The panel header turns green on `CI_PASSED`, red on `CI_FAILED`.
    - Rejection-log strip (only visible if the run has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, PARSING=blue, PARSED=blue, PLANNING=yellow, PLANNED=yellow, DRAFTING=blue, DRAFTED=blue, AWAITING_REVIEW=indigo, CI_RUNNING=yellow, CI_PASSED=green, MERGE_READY=green, CI_FAILED=red, REJECTED=red, FAILED=red.

Each phase panel renders only when its data is present on the row record. A run in `PLANNING` shows only panel 1 (with a spinner if the agent has not yet returned). A run in `AWAITING_REVIEW` shows all three panels plus the reviewer action bar. This is the visual proof that the typed handoff between phases is the only path information travels.
