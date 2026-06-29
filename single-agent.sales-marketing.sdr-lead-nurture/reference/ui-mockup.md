# UI mockup — sdr-lead-nurture

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SDR Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `SDR <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **Ingest lead** and fill in the form, or click **Load seeded example** to pre-fill with one of the four seeded leads.
  3. Click **Start engagement** and watch the card enter `SANITIZED` then `ENGAGING`.
  4. Reply as the lead to progress the conversation; the agent responds each turn until it books a meeting, dismisses, or hands off.
- Card **How it works**: one paragraph on receive → sanitize → engage → close; one paragraph on the three governance mechanisms (PII sanitizer, brand-safety guardrail, write-action guardrail).
- Card **Components**: rows per component (LeadEntity, LeadSanitizer, LeadWorkflow, SdrAgent, ReplyGuardrail, BookingGuardrail, QualityScorer, LeadView, LeadEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 2 Guardrails + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `pii: true` in Data is filled and prominent. `decisions.authority_level = autonomous-with-guardrails` and `oversight.human_on_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` render faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, G2). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), G2 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ingest a lead. <span class="accent">Watch it engage.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Ingest panel + live list.
    - Ingest panel: fields for First name, Last name, Company, Job title, Email, Phone (optional), Channel dropdown (WEBSITE_CHAT / EMAIL / EVENT / REFERRAL), Initial message textarea; a "Load seeded example" link that fills all fields from the seeded lead corpus; and a teal `Start engagement` button.
    - Live list below the panel: one card per lead, newest-first. Each card shows: status pill, lead name + company, age, and — once closed — decision chip (MEETING_BOOKED / DISMISSED / HANDED_OFF).
  - **Right column** — Selected-lead detail.
    - Header: status pill + decision chip (if closed) + quality score chip (if scored) + lead name + company.
    - Contact summary: job title, channel badge, redacted email/phone (from sanitized record).
    - Sanitized record preview: monospace block of the redacted JSON, with PII category chips above (`email`, `phone`, `person-name`, etc.).
    - Conversation thread: alternating LEAD / AGENT bubbles in chronological order. LEAD bubbles left-aligned; AGENT bubbles right-aligned.
    - Reply composer: a text input + **Send** button, visible only while status is `ENGAGING`.
    - Booking card (visible after `MEETING_BOOKED`): slot datetime, duration, AE id, Zoom link.
    - Quality section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border orange.
- Status pill colours: RECEIVED=muted, SANITIZED=blue, ENGAGING=yellow, MEETING_BOOKED=green, DISMISSED=muted, HANDED_OFF=blue, FAILED=red.
- Decision chip colours: MEETING_BOOKED=green, DISMISSED=muted, HANDED_OFF=blue.

The raw email and phone number are never displayed on this screen — only the redacted tokens. The full contact record (including raw PII) is accessible via `GET /api/leads/{id}` and lives only in the entity log for audit purposes.
