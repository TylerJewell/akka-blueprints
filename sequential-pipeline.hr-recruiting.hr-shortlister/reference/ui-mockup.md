# UI mockup — hr-shortlister

Five-tab structure. Browser title: `<title>Akka Sample: HR Candidate Shortlister</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation (Lesson 26):

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `HR Candidate <span class="accent">Shortlister</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded applicant profiles (or paste your own resume text), select a job, and click **Submit application**.
  3. Watch the card transition through PARSING → PARSED → SCORING → SCORED → DECIDING → DECIDED → AWAITING_APPROVAL.
  4. Click **Approve** (or override the decision) to advance to APPROVED → WRITTEN.
- Card **How it works**: one paragraph on the three task phases (PARSE → SCORE → DECIDE) and the typed handoff between them; one paragraph on the three governance mechanisms (special-category sanitizer, recruiter approval gate, fairness drift monitor).
- Card **Components**: rows per component (ApplicationEntity, DriftReportEntity, ShortlistingWorkflow, FairnessDriftWorkflow, ShortlistAgent, ParseTools, ScoreTools, ShortlistTools, SpecialCategoryGuardrail, FairnessScorer, ErpNextAdapter, ApplicationView, ShortlistEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 2 Workflows, 2 EventSourcedEntities, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail/Sanitizer, 1 Scorer, 1 Adapter as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` and `special-category-employment: true` declarations in Data are filled (name, email, resume text are PII; resumes may contain age/gender/nationality). `decisions.authority_level = gated-human` and `oversight.human_in_loop = true` are the distinctive answers. `automated-employment-decision: true` in Compliance is highlighted. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, H1, E1). ID badges coloured: S1 orange (sanitizer), H1 purple (hitl), E1 blue (eval-periodic).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.
- Below the table, a **Fairness Drift** panel that fetches `GET /api/drift-reports` and renders the current `DriftReport` per cohort dimension. A breached report (`breached: true`) renders with a red border and the ratio displayed prominently. A non-breached report renders with a green border.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a resume. <span class="accent">See the shortlist decision.</span>`. Subtitle: `One agent, three task phases, one sanitizer, one approval gate.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a textarea `Resume text` (with a "Pick a seeded profile" dropdown that fills it), a select `Job` (populated from the three seeded job configs), and a yellow `Submit application` button.
    - Live list below: one card per application, newest-first. Each card shows status pill, applicant name, job id, age, and a small orange dot if any special-category redaction fired.
  - **Right column** — Selected-application detail.
    - Header: status pill + applicant name + job id.
    - Phase panel 1 (Parsed profile): a table with columns field, value. Special-category fields that were redacted show `[REDACTED]` with an orange chip. Visible once `profile` is present.
    - Phase panel 2 (Score): a per-criterion table (criterion name, score 0–10, justification) and overall score bar (0–100). Visible once `score` is present.
    - Phase panel 3 (Agent decision): decision badge (`SHORTLIST` / `HOLD` / `REJECT`), confidence score, rationale paragraph. Visible once `agentDecision` is present.
    - Approval panel (only when `status == AWAITING_APPROVAL`): a prominent yellow banner with the agent's decision displayed, two buttons (`Approve` and `Override`). The Override path reveals a Decision dropdown and a Rationale textarea.
    - Recruiter decision section (once `recruiterDecision` is present): recruiter id, final decision, override rationale (if any), timestamp. Shows "Approved as-is" or "Overridden" badge.
    - ERPNext write confirmation (once `status == WRITTEN`): the ERPNext record id returned by the adapter stub.
    - Redaction-log strip (only visible if `redactions` is non-empty): a small table with field name, original presence, and timestamp.
- Status pill colours: RECEIVED=muted, PARSING=blue, PARSED=blue, SCORING=yellow, SCORED=yellow, DECIDING=yellow, DECIDED=yellow, AWAITING_APPROVAL=orange, APPROVED=purple, WRITTEN=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. An application in `SCORING` shows panel 1 (profile) with the redaction chips already visible, while panel 2 shows a spinner. This is the visual proof that the PARSE phase's typed `Profile` output is the only path resume content travels to the SCORE phase.
