# UI mockup — hrsd-inquiry

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: HRSD Inquiry Agent</title>`.

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
- Headline: `HRSD Inquiry <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a topic (Benefits / Leave & Absence / Onboarding / Payroll / General HR) and type a question, or load a seeded example.
  3. Click **Ask HR**.
  4. Watch the card transition through SCREENED → ANSWERING → ANSWERED (and optionally REQUEST_SUBMITTED).
- Card **How it works**: one paragraph on submit → screen → answer → optional-request-submit; one paragraph on the two governance mechanisms (special-category screener, policy-citation guardrail).
- Card **Components**: rows per component (InquiryEntity, SpecialCategoryScreener, InquiryWorkflow, HrInquiryAgent, ResponseGuardrail, HrRequestSubmitter, InquiryView, InquiryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Submitter as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `special_category_data: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Get a policy answer.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Topic` (Benefits / Leave & Absence / Onboarding / Payroll / General HR), `Employee ID` text input, `Message` textarea (with a "Load seeded example" link that fills both the topic and the message), a checkbox `Submit HR service request if applicable`, and a yellow `Ask HR` button.
    - Live list below: one card per inquiry, newest-first. Each card shows status pill, topic badge, age, and — when answered — a one-line preview of the answer.
  - **Right column** — Selected-inquiry detail.
    - Header: status pill + topic badge + `employeeId`.
    - Screened message: a monospace block of the redacted message with special-category category chips above (e.g. `health-condition`, `union-membership`).
    - Cited policies: a list of policy id chips, each chip showing `policyId` and `title`.
    - Answer: the agent's 1–4-sentence paragraph.
    - Service request section (when present): `requestType` badge, `description` text, key-value pairs table.
    - Confirmation reference (when request submitted): a green chip showing the `HR-xxxxx-xxxxx` reference number.
- Status pill colours: SUBMITTED=muted, SCREENED=blue, ANSWERING=yellow, ANSWERED=blue, REQUEST_SUBMITTED=green, FAILED=red.
- Topic badge colours: BENEFITS=purple, LEAVE_AND_ABSENCE=blue, ONBOARDING=teal, PAYROLL=orange, GENERAL_HR=muted.

The raw message is never displayed on this screen — only the screened form. HR representatives who need the raw text fetch `/api/inquiries/{id}` and read `request.rawMessage` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
