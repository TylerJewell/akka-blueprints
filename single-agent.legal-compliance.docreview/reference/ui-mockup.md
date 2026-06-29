# UI mockup — docreview

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: DocReview</title>`.

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

And the DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents (clicking a tab and seeing a blank because a hidden zombie panel occupied the index).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Doc<span class="accent">Review</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded instruction set (DPA / MSA / OSS-notice) and paste the matching seed document, or load both with one click.
  3. Click **Submit for review**.
  4. Watch the card transition through SANITIZED → REVIEWING → VERDICT_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → sanitize → review → eval; one paragraph on the three governance mechanisms (sanitizer, guardrail, on-decision eval).
- Card **Components**: rows per component (ReviewEntity, DocumentSanitizer, ReviewWorkflow, DocumentReviewerAgent, VerdictGuardrail, EvaluationScorer, ReviewView, ReviewEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, E1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a document. <span class="accent">Read the verdict.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Instruction set` (DPA / MSA / OSS-notice / custom), `Document title` text input, `Document` textarea (with a "Load seeded example" link that fills both title and body), `Submitted by` text input, and a yellow `Submit for review` button.
    - Live list below: one card per review, newest-first. Each card shows status pill, decision badge (when verdict landed), eval score chip (when eval landed), document title, age.
  - **Right column** — Selected-review detail.
    - Header: status pill + decision badge + eval score chip + `documentTitle`.
    - Submitted instructions: a small list with `instructionId`, `severityFloor` chip, and the instruction text.
    - Sanitized document preview: a monospace block of the redacted document, with PII category chips above (`email`, `phone`, `ssn`, etc.).
    - Verdict summary: the agent's 1–3-sentence paragraph.
    - Findings table: columns instruction id, severity (coloured chip), document section, quote (italic), recommendation.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, REVIEWING=yellow, VERDICT_RECORDED=blue, EVALUATED=green, FAILED=red.
- Decision badge colours: PASS=green, NEEDS_REVISION=yellow, FAIL=red.
- Severity chip colours: LOW=muted, MEDIUM=blue, HIGH=yellow, CRITICAL=red.

The raw document is never displayed on this screen — only the sanitized form. Reviewers who need the raw text fetch `/api/reviews/{id}` and read `request.rawDocument` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
