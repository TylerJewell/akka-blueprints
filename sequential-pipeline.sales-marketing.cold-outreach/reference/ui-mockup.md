# UI mockup — cold-outreach

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: LeadPilot Lite - Personalized Cold Email</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `LeadPilot Lite - <span class="accent">Personalized Cold Email</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded prospects (or enter a company name and email) and click **Run outreach**.
  3. Watch the card transition through RESEARCHING → RESEARCHED → DRAFTING → DRAFTED → AWAITING_REVIEW. When the yellow banner appears, click **Approve** (or **Reject**).
  4. After approval, watch the card advance to SENDING → SENT. Inspect the compliance badge and rejection-log strip if any guardrail fired.
- Card **How it works**: one paragraph on the three task phases (RESEARCH → DRAFT → SEND) and the typed handoff between them; one paragraph on the three governance mechanisms (send-gate guardrail, compliance guardrail, HITL approval gate).
- Card **Components**: rows per component (ProspectEntity, OutreachPipelineWorkflow, OutreachAgent, ResearchTools, DraftTools, SendTools, SendGuardrail, ComplianceGuardrail, ProspectView, OutreachEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 2 Guardrails, and 1 HITL gate as supporting elements).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled (contact emails are PII). `decisions.authority_level = execute` and `oversight.human_in_loop = true` are the distinctive answers. The `can-spam-compliant-unsubscribe-mechanism: true` compliance field is pre-filled. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H2, HITL1). ID badge colours: G1 red (guardrail · before-tool-call), H2 orange (guardrail · before-agent-response), HITL1 yellow (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Enter a prospect. <span class="accent">Approve the draft.</span>`. Subtitle: `One agent, three task phases, one human gate between draft and send.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Company name` + text input `Contact email` (with a "Pick a seeded prospect" dropdown that fills both), and a yellow `Run outreach` button.
    - Live list below: one card per prospect, newest-first. Each card shows status pill, company name, contact email, age, a green compliance badge when compliance passed, and a small red dot if any guardrail rejection fired during this prospect's pipeline.
  - **Right column** — Selected-prospect detail.
    - Header: status pill + company name + contact email.
    - Phase panel 1 (Research): firmographic summary (industry, employee count, country) + intent-signals table (signalType, description, observedAt). Visible once `profile.isPresent()`.
    - Phase panel 2 (Email draft): subject line, body text (pre-formatted), personalization fields list, compliance badge (PASSED / FAILED with rule list). Visible once `draft.isPresent()`.
    - HITL approval panel: yellow banner "Awaiting reviewer approval" with **Approve** button (green) and **Reject** button (red). Input field for reviewer note. Active only when `status == AWAITING_REVIEW`. After decision, shows the reviewer's note and timestamp.
    - Phase panel 3 (Sent email): subject, recipient, sent timestamp, message ID. Visible once `outreachEmail.isPresent()`.
    - Rejection-log strip (only visible if the prospect has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, RESEARCHING=blue, RESEARCHED=blue, DRAFTING=yellow, DRAFTED=yellow, AWAITING_REVIEW=yellow-pulse, SENDING=blue, SENT=green, REVIEW_REJECTED=red, FAILED=red.

Each phase panel renders only when its data is present on the row record. A prospect in `DRAFTING` shows panel 1 and a spinner for panel 2. This is the visual proof that the typed handoff between phases is the only path information travels. The HITL approval panel is the only interactive element in the right column that modifies state (via `POST /api/outreach/{id}/review`); all other right-column content is read-only.
