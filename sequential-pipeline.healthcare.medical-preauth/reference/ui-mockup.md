# UI mockup — medical-preauth

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Medical Pre-Authorization</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation:

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
- Headline: `Medical <span class="accent">Pre-Authorization</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded requests (or fill in a procedure code) and click **Submit request**.
  3. Watch the card transition through VALIDATING → VALIDATED → REVIEWING → POLICY_REVIEWED → DETERMINING → DETERMINED → EVALUATED (or PENDING_HUMAN_REVIEW for a DENIED outcome).
  4. If the request lands in PENDING_HUMAN_REVIEW, click **Acknowledge denial** to finalize it.
- Card **How it works**: one paragraph on the three task phases (VALIDATE → REVIEW → DETERMINE) and the typed handoff between them; one paragraph on the three governance mechanisms (PHI sanitizer, denial human hold, clinical completeness eval).
- Card **Components**: rows per component (PreAuthEntity, PreAuthWorkflow, PreAuthAgent, ValidateTools, ReviewTools, DetermineTools, PhiSanitizer, ClinicalCompletenessScorer, PreAuthView, PreAuthEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Sanitizer/Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `phi: true` declaration in Data is filled (member IDs, clinical notes). `decisions.authority_level = binding` and `oversight.human_in_loop = true` are the distinctive answers. `denial_hold_timeout_hours: 72` is highlighted. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, H1, E1). ID badges coloured: S1 orange (sanitizer), H1 amber (hitl), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a request. <span class="accent">Track the determination.</span>`. Subtitle: `One agent, three task phases, PHI stripped at every tool call.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a "Pick a seeded request" dropdown (fills all fields at once) OR individual text inputs for Member ID, Procedure Code (CPT), Diagnosis Code (ICD-10), Physician NPI, and Clinical Notes. A blue **Submit request** button.
    - Live list below: one card per request, newest-first. Each card shows status pill, outcome chip (APPROVED/DENIED/PENDING when available), procedure code, age, a small orange dot if any PHI sanitization fired, and a small red dot if any phase-gate rejection fired.
  - **Right column** — Selected-request detail.
    - Header: status pill + outcome chip + procedure code + diagnosis code.
    - Phase panel 1 (Validation): eligibility row (plan type, coverage tier, active) and procedure row (code recognized, ICD-10 linked). Visible once `validationResult` is present.
    - Phase panel 2 (Policy review): matched articles table (articleId, title, policyVersion) and criteria table (criterionId, description, met/unmet badge, evidence). Visible once `policyReview` is present.
    - Phase panel 3 (Determination): outcome badge (green APPROVED / red DENIED / amber PENDING_ADDITIONAL_INFO), rationale paragraph, cited articles chips, covered procedure code chips. Visible once `determination` is present.
    - Eval section: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - **Acknowledge denial** button (only visible when `status == PENDING_HUMAN_REVIEW`): posts `{ reviewerNotes }` to the acknowledge endpoint. Button changes to "Acknowledged" on success.
    - Sanitization-log strip (visible if any `SanitizationApplied` events exist): table with field, hashedToken, tool, time.
    - Phase-rejection-log strip (visible if any `PhaseGuardrailRejected` events exist): table with phase, tool, reason, time.
- Status pill colours: SUBMITTED=muted, VALIDATING=blue, VALIDATED=blue, REVIEWING=yellow, POLICY_REVIEWED=yellow, DETERMINING=blue, DETERMINED=blue, PENDING_HUMAN_REVIEW=amber, EVALUATED=green, ESCALATED=red, FAILED=red.

Each phase panel renders only when its data is present on the row record. A request in `REVIEWING` shows validation panel only (the review panel shows an in-progress spinner). A request in `PENDING_HUMAN_REVIEW` shows all three phase panels plus the **Acknowledge denial** button — the determination is visible for the reviewer's inspection.
