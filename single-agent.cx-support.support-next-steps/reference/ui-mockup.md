# UI mockup — support-next-steps

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SupportNextSteps</title>`.

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
- Headline: `Support<span class="accent">NextSteps</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded ticket (Billing / Authentication / Data Export) and click **Load seeded example** to fill the subject and body.
  3. Click **Submit ticket**.
  4. Watch the card transition through SANITIZED → ADVISING → RECOMMENDATION_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → sanitize → advise → eval; one paragraph on the two governance mechanisms (PII sanitizer, on-decision eval).
- Card **Components**: rows per component (TicketEntity, TicketSanitizer, TicketWorkflow, ResolutionAdvisorAgent, RecommendationScorer, TicketView, TicketEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Scorer as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, E1). ID badges coloured: S1 green (sanitizer), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a ticket. <span class="accent">Read the recommendation.</span>`. Subtitle: `One agent, grounded in past resolutions.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Product area` (Billing / Authentication / Data Export), `Subject` text input, `Ticket body` textarea (with a "Load seeded example" link that fills both subject and body), `Submitted by` text input, and a yellow `Submit ticket` button.
    - Live list below: one card per ticket, newest-first. Each card shows status pill, confidence badge (when recommendation landed), eval score chip (when eval landed), subject line, age.
  - **Right column** — Selected-ticket detail.
    - Header: status pill + confidence badge + eval score chip + subject.
    - Sanitized ticket body: a monospace block of the redacted body, with PII category chips above (`email`, `phone`, `account`, etc.).
    - Recommendation rationale: the agent's 1–2-sentence paragraph.
    - Steps table: columns step number, description, action type chip, confidence chip, resolution reference (linked or plain text).
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, ADVISING=yellow, RECOMMENDATION_RECORDED=blue, EVALUATED=green, FAILED=red.
- Confidence badge colours: HIGH=green, MEDIUM=yellow, LOW=muted.
- Action type chip colours: VERIFY=blue, ESCALATE=red, CONFIGURE=purple, INFORM=muted, INVESTIGATE=yellow.

The raw ticket body is never displayed on this screen — only the sanitized form. Support agents who need the raw text fetch `/api/tickets/{id}` and read `request.ticketBody` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
