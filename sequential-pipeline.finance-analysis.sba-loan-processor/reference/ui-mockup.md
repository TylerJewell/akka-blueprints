# UI mockup — sba-loan-processor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Small Business Loan Agent</title>`.

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
- Headline: `Small Business <span class="accent">Loan Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded loan application from the dropdown and click **Process application**.
  3. Watch the card transition through INTAKE → UNDERWRITING → DECISION → PENDING_REVIEW.
  4. Click **Approve** or **Deny** in the officer review panel to complete the application.
- Card **How it works**: one paragraph on the four task phases (INTAKE → UNDERWRITE → DECIDE → REPORT) and the typed handoff between them; one paragraph on the four governance mechanisms (PII sanitizer, protected-attribute sanitizer, human-in-the-loop officer gate, fair-lending eval).
- Card **Components**: rows per component with Kind column coloured by Akka primitive type — LoanApplicationEntity (EventSourcedEntity), LoanPipelineWorkflow (Workflow), LoanUnderwritingAgent (AutonomousAgent), IntakeTools / UnderwriteTools / DecisionTools / ReportTools (function-tool classes), PiiSanitizer / ProtectedAttributeSanitizer (sanitizers), PhaseGuardrail (guardrail), FairLendingScorer (scorer), LoanApplicationView (View), LoanEndpoint / AppEndpoint (HttpEndpoints).
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 4 function-tool classes, 2 Sanitizers, 1 Guardrail, 1 Scorer).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` and `special-category: true` declarations in Data are prominent. `decisions.authority_level = human-final` and `oversight.human_in_loop = true` are the distinctive answers for this domain. `ecoa_compliance_review_required: true` and `adverse_action_notice_required: true` appear under Compliance. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with four rows (S1, S2, H1, E1). ID badge colours: S1 and S2 purple (sanitizer), H1 amber (human-in-the-loop), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an application. <span class="accent">Review the decision.</span>`. Subtitle: `One agent, four task phases, two sanitizers, and a human gate between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a "Pick a seeded application" dropdown that fills applicant reference, loan amount, and loan purpose fields; and a yellow **Process application** button.
    - Live list below: one card per application, newest-first. Each card shows: status pill, fairness eval score chip (when eval landed), applicant reference, loan amount, age, and a small amber dot if the application is in `PENDING_REVIEW`, or a small red dot if any guardrail rejection fired.
  - **Right column** — Selected-application detail.
    - Header: status pill + fairness eval score chip + applicant reference + loan amount.
    - Phase panel 1 (Credit profile): credit score tier badge (colour-coded by tier), score, business name, industry, years in operation. Visible once `creditProfile` is present.
    - Phase panel 2 (Underwriting analysis): annual revenue, operating expenses, NOI, collateral type and value, LTV ratio, key risk factors list, mitigants list. Visible once `analysis` is present.
    - Phase panel 3 (Loan decision): recommendation badge (APPROVE=green / APPROVE_WITH_CONDITIONS=amber / DENY=red), proposed amount, interest-rate tier, DSCR value, risk tier badge, decision rationale text, conditions list. Visible once `decision` is present.
    - Fairness eval section: a 1–5 score badge, flags list (empty on happy path), one-line rationale. Score ≤ 2 highlights the section border amber. Visible once `fairnessEval` is present.
    - Officer review panel: two buttons — **Approve** (green) and **Deny** (red) — with a notes textarea and an officer ID field. Visible only when `status == PENDING_REVIEW`. Sends `POST /api/loans/{id}/review`. Disabled after submission.
    - Rejection-log strip: a small table with phase, tool, reason, time. Visible only if the application has any `guardrailRejections`.
- Status pill colours: SUBMITTED=muted, INTAKE_IN_PROGRESS=blue, INTAKE_COMPLETE=blue, UNDERWRITING=yellow, UNDERWRITING_COMPLETE=yellow, DECISION_IN_PROGRESS=blue, DECISION_MADE=blue, PENDING_REVIEW=amber, OFFICER_APPROVED=green, OFFICER_DENIED=red, FAILED=red.

Each phase panel renders only when its data is present on the row record. An application in `UNDERWRITING` shows only the credit-profile panel (with the underwriting panel showing a spinner). An application in `PENDING_REVIEW` shows all three phase panels plus the eval section plus the officer review panel. This is the visual proof that typed handoffs are the only path information travels between phases.
