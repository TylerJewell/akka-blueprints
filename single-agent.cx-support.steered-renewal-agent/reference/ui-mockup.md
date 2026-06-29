# UI mockup — steered-renewal-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Library Book Renewal (Steering)</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Library Book <span class="accent">Renewal</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded patron and loan from the dropdowns, or enter a custom patron ID.
  3. Click **Request renewal**.
  4. Watch the card transition through ENRICHED → DECIDING → DECISION_RECORDED → COMPLETED.
- Card **How it works**: one paragraph on request → enrich → decide → notify; one paragraph on the steering guardrail (before-agent-response) and what it enforces.
- Card **Components**: rows per component (LoanEntity, RenewalWorkflow, RenewalAgent, PolicyEnforcer, RenewalView, RenewalEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = enforced` declaration is prominent — this is not an advisory system. `oversight.human_in_loop = false` is the distinctive answer; `human_on_loop = true` notes that administrators can review decisions after the fact. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Request a renewal. <span class="accent">See the decision.</span>`. Subtitle: `One agent, one steering guardrail.`
- Layout: two-column.
  - **Left column** — Request form + live list.
    - Request form: `Patron` dropdown (seeded patrons: Alex Rivera / Morgan Chen / Sam Torres or custom), `Loan` dropdown (updates based on selected patron, showing item title + due date), and a yellow `Request renewal` button.
    - Live list below: one card per renewal, newest-first. Each card shows status pill, outcome badge (when decision landed), patron display name, item title, age.
  - **Right column** — Selected-renewal detail.
    - Header: status pill + outcome badge + `itemTitle`.
    - Patron summary: name, tier chip, outstanding fines (highlighted red if above threshold).
    - Loan detail: item type, original due date, prior renewal count / max renewals allowed.
    - Decision section: outcome badge, reason sentence, new due date (if present).
    - Notification section: channel chip, message text, sent-at timestamp.
- Status pill colours: REQUESTED=muted, ENRICHED=blue, DECIDING=yellow, DECISION_RECORDED=blue, COMPLETED=green, FAILED=red.
- Outcome badge colours: APPROVED=green, EXTENDED=blue, DENIED=red.
- Patron tier chip colours: STANDARD=muted, PREMIUM=blue, STAFF=yellow.
