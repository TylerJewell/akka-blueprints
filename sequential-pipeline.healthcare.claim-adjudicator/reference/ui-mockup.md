# UI mockup — claim-adjudication-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Claim Adjudication Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Claim Adjudication <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded claims (or fill in a custom claim) and click **Submit claim**.
  3. Watch the card transition through VALIDATING → VALIDATED → EVALUATING → EVALUATED_COVERAGE → DECIDING → DECIDED.
  4. If the claim is denied, use the reviewer action panel on the right to approve or override. Watch it reach ADJUDICATION_EVALUATED.
- Card **How it works**: one paragraph on the three task phases (VALIDATE → EVALUATE → ADJUDICATE) and the typed handoff between them; one paragraph on the three governance mechanisms (PHI sanitiser, denial human-review hold, adjudication completeness eval).
- Card **Components**: rows per component (ClaimEntity, ClaimAdjudicationWorkflow, AdjudicationAgent, ValidateTools, EvaluateTools, AdjudicateTools, PhiSanitizerGuardrail, AdjudicationScorer, ClaimView, ClaimEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail/Sanitiser, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `phi: true` declaration in Data is filled — claims contain Protected Health Information. `decisions.authority_level = binding` and `oversight.human_in_loop = true` (denial review required) are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, H1, E1). ID badge colours: S1 orange (sanitiser), H1 yellow (human-in-the-loop), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a claim. <span class="accent">Review the decision.</span>`. Subtitle: `One agent, three task phases, PHI sanitised before every tool call.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a "Pick a seeded claim" dropdown (LAB-001, SURG-002, MH-003, DENIED-004) that fills form fields, plus manual-entry fields (member plan, procedure codes, service date). A teal **Submit claim** button.
    - Live list below: one card per claim, newest-first. Each card shows status pill, outcome badge (APPROVED / DENIED / PENDING_REVIEW when decided), procedure family, age, and a small orange dot if any PHI was redacted during processing.
  - **Right column** — Selected-claim detail.
    - Header: status pill + outcome badge + procedure family label.
    - Phase panel 1 (Validation): eligibility status chip, procedure code table (code, system, recognised, family). Visible once `validation.isPresent()`.
    - Phase panel 2 (Coverage): applicable rules list (ruleId, description, exclusion flag) and coverage amount summary. Visible once `coverage.isPresent()`.
    - Phase panel 3 (Decision): outcome badge, rationale narrative, cited rule IDs, approved amount or denial code. Visible once `decision.isPresent()`.
    - Reviewer action panel (only visible when `status == PENDING_REVIEW`): shows decision summary and denial code; **Approve denial** button and **Override to approve** button with a notes textarea and reviewer ID field.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red and adds a "secondary review" badge.
    - PHI-redaction log strip (always visible): a small table with field name, masked value, and timestamp. Empty on claims with no PHI fields processed.
    - Rejection-log strip (only visible if `guardrailRejections` is non-empty): phase, tool, reason, time.
- Status pill colours: CREATED=muted, VALIDATING=blue, VALIDATED=blue, EVALUATING=yellow, EVALUATED_COVERAGE=yellow, DECIDING=blue, DECIDED=blue, PENDING_REVIEW=orange, APPROVED=green, DENIED=red, ADJUDICATION_EVALUATED=green, ESCALATED=purple, FAILED=red.

Each phase panel renders only when its data is present on the row record. A claim in `EVALUATING` shows panel 1 (validation present) and panel 2 (in-progress spinner). This is the visual proof that the typed handoff between phases is the only path information travels.
