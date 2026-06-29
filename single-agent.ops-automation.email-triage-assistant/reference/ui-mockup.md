# UI mockup — email-triage-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Multi-Modal Email Assistant</title>`.

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
- Headline: `Multi-Modal Email <span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded email (billing dispute / product support / contract review) and click **Load seeded example**, or paste your own email body.
  3. Click **Submit for triage**.
  4. Read the draft reply, then click **Approve** or **Reject** to control whether the email is sent.
- Card **How it works**: one paragraph on submit → sanitize → triage → confirm → send; one paragraph on the three governance mechanisms (PII sanitizer, before-tool-call guardrail, human-in-the-loop confirmation).
- Card **Components**: rows per component (EmailEntity, ConfirmationEntity, EmailSanitizer, TriageWorkflow, EmailTriageAgent, SendGuardrail, TriageView, TriageEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = assist-and-confirm` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, H1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), H1 yellow (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Triage an email. <span class="accent">Approve the draft.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded email` (Billing Dispute / Product Support / Contract Review / custom), `From address` text input, `Subject` text input, `Email body` textarea (with a "Load seeded example" link that fills all fields), `Attachment text` textarea (optional), `Submitted by` text input, and a yellow `Submit for triage` button.
    - Live list below: one card per email, newest-first. Each card shows status pill, urgency badge (when triage lands), category chip (when triage lands), subject truncated, age.
  - **Right column** — Selected-email detail.
    - Header: status pill + urgency badge + category chip + subject.
    - Sanitized body preview: a monospace block of the redacted email body, with PII category chips above (`email`, `phone`, `person-name`, `account-id`, etc.).
    - Attachment section (if present): a monospace block of the redacted attachment text.
    - Triage classification: urgency badge, category chip, and the `classificationRationale` paragraph.
    - Draft reply: a text block with the full draft text, formatted as a readable email.
    - Confirm section (visible when status is `DRAFT_READY`): a green **Approve** button and a red **Reject** button side by side. On click, POSTs to `/api/emails/{id}/confirm`. Both buttons are disabled after either is clicked to prevent double-submission.
    - Status notes: when status is `SENT`, show "Draft approved and sent." When status is `REJECTED`, show "Draft rejected. No email was sent."
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, REVIEWING=yellow, DRAFT_READY=purple, SENDING=yellow, SENT=green, REJECTED=red, FAILED=red.
- Urgency badge colours: HIGH=red, MEDIUM=yellow, LOW=muted.
- Category chip colours: BILLING=blue, SUPPORT=green, LEGAL=purple, GENERAL=muted.

The raw email body and raw attachment text are never displayed on this screen — only the sanitized forms. Operators who need the raw text fetch `GET /api/emails/{id}` and read `submission.rawBody` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
