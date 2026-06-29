# UI mockup — service-agent-omnichannel

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Service Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Service <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded channel and scenario (billing dispute / technical fault / returns request / account closure) and click **Load seeded example**.
  3. Click **Send message**.
  4. Watch the card transition through SANITIZED → TRIAGING → HANDLING → REPLIED (or ESCALATED).
- Card **How it works**: one paragraph on receive → sanitize → triage → handle → eval; one paragraph on the four governance mechanisms (PII sanitizer, reply guardrail, CRM-write guardrail, HITL escalation).
- Card **Components**: rows per component (CaseEntity, MessageSanitizer, CaseWorkflow, ServiceAgent, ReplyGuardrail, CrmWriteGuardrail, CaseTriage, EscalationScorer, CaseView, CaseEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 2 Guardrails + 1 Triage + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = act-on-behalf` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with four rows (S1, G1, G2, H1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), G2 red (guardrail), H1 amber (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Read the reply.</span>`. Subtitle: `One agent, four governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Channel` (WhatsApp / Voice / Web / Facebook), dropdown `Scenario` (billing dispute / technical fault / returns request / account closure / custom), `Customer ID` text input, `Customer message` textarea (with a **Load seeded example** link that fills all fields), and a yellow **Send message** button.
    - Live list below: one card per case, newest-first. Each card shows status pill, resolution badge (when reply landed), eval score chip (when eval landed), channel chip, scenario label, age.
  - **Right column** — Selected-case detail.
    - Header: status pill + resolution badge + eval score chip + channel chip + `scenario`.
    - Sanitized message preview: a monospace block of the redacted message, with PII category chips above (`email`, `phone`, `account-number`, etc.). Raw message is never shown here.
    - Triage category chip.
    - Reply section: the agent's `replyText` in a blockquote styled for the channel format; `channelFormat` badge; `crmWritesApplied` list.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - If status is `ESCALATED`: a distinct amber human-queue chip replaces the resolution badge, and a note reads "Transferred to human agent queue."
- Status pill colours: RECEIVED=muted, SANITIZED=blue, TRIAGING=blue, HANDLING=yellow, REPLIED=blue, ESCALATED=amber, RESOLVED=green, FAILED=red.
- Resolution badge colours: RESOLVED=green, NEEDS_FOLLOW_UP=yellow, ESCALATE=amber.
- Channel chip colours: WHATSAPP=green, VOICE=purple, WEB=blue, FACEBOOK=indigo.

The raw customer message is never displayed on this screen — only the sanitized form. Operators who need the raw text fetch `/api/cases/{id}` and read `message.rawMessage` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the original.
