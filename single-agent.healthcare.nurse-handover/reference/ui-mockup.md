# UI mockup — nurse-handover

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: NurseHandover</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Nurse<span class="accent">Handover</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded ward (General Medicine / ICU / Paediatrics) and load the matching seed report.
  3. Fill in your name and click **Submit handover**.
  4. Watch the card transition through SANITIZED → SUMMARIZING → SUMMARY_READY → AWAITING_SIGNOFF, then click **Sign off** to seal it.
- Card **How it works**: one paragraph on submit → sanitize → summarize → await-signoff; one paragraph on the two governance mechanisms (PHI sanitizer, clinical sign-off gate).
- Card **Components**: rows per component (HandoverEntity, ReportSanitizer, HandoverWorkflow, HandoverAgent, HandoverView, HandoverEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `phi: true` declaration in Data is filled and prominent. `oversight.human_in_loop = true` and `decisions.authority_level = recommend-only` are the distinctive answers. `hipaa_baa_with_llm_provider` is shown as a required deployer field. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, H1). ID badges coloured: S1 green (sanitizer), H1 amber (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a handover. <span class="accent">Sign it off.</span>`. Subtitle: `One agent, two governance controls around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Ward` (General Medicine / ICU / Paediatrics), `Shift date` date input, `Submitted by` text input, `Shift report` textarea (with a "Load seeded example" link that fills both the textarea and the ward dropdown), and a teal `Submit handover` button.
    - Live list below: one card per handover, newest-first. Each card shows status pill, ward badge, age, and a **Sign off** button when the handover is in `AWAITING_SIGNOFF`.
  - **Right column** — Selected-handover detail.
    - Header: status pill + ward badge + `shiftDate` + `submittedBy`.
    - Checklist preview: a small list with `itemId`, `urgency` chip, and the description.
    - Sanitized report preview: a monospace block of the redacted report, with PHI category chips above (`mrn`, `dob`, `phone`, etc.).
    - Per-patient status table: columns patient ref, current condition, outstanding tasks, risk level (coloured chip).
    - Outstanding tasks list: ordered CRITICAL → URGENT → ROUTINE, with urgency chip per row.
    - Risk flags: highlighted amber box listing any flags.
    - Narrative summary: 2–4-sentence paragraph from the agent.
    - Sign-off section at the bottom: if `AWAITING_SIGNOFF`, show a `Signed off by` text input and a green `Sign off` button. If `SIGNED_OFF`, show `Signed off by: <name> at <time>` in muted text.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, SUMMARIZING=yellow, SUMMARY_READY=blue, AWAITING_SIGNOFF=amber, SIGNED_OFF=green, FAILED=red.
- Risk level chip colours: LOW=muted, MODERATE=blue, HIGH=yellow, CRITICAL=red.
- Urgency chip colours: ROUTINE=muted, URGENT=yellow, CRITICAL=red.

The raw shift report is never displayed on this screen — only the sanitized form. Clinicians who need the raw text fetch `/api/handovers/{id}` and read `request.rawReport` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
