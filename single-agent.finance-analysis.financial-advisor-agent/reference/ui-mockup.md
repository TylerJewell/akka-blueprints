# UI mockup — financial-advisor-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Financial Advisor Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Financial<span class="accent">AdvisorAgent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded client profile (IRA conservative / Brokerage moderate / ROTH_IRA aggressive) and load the matching question with one click.
  3. Click **Ask advisor**.
  4. Watch the card transition through SANITIZED → ADVISING → RESPONSE_RECORDED → AUDITED.
- Card **How it works**: one paragraph on submit → sanitize → advise → audit; one paragraph on the two governance mechanisms (sector sanitizer, before-agent-response guardrail).
- Card **Components**: rows per component (AdvisoryEntity, ClientDataSanitizer, AdvisoryWorkflow, FinancialAdvisorAgent, RecommendationGuardrail, AdvisoryView, AdvisoryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` and `financial-account-identifiers: true` declarations in Data are filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic to indicate deployer responsibility.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask an investment question. <span class="accent">Read the advisory.</span>`. Subtitle: `One agent, sector sanitizer and response guardrail around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Client profile` (IRA conservative / Brokerage moderate / ROTH_IRA aggressive / custom), `Investment question` textarea (with a "Load seeded example" link that fills the profile and question), and a yellow `Ask advisor` button.
    - Live list below: one card per advisory, newest-first. Each card shows status pill, risk rating badge (when response landed), document title (question excerpt), age.
  - **Right column** — Selected-advisory detail.
    - Header: status pill + risk rating badge + question excerpt.
    - Client profile summary: account type chip, risk tolerance chip, holding count.
    - Sanitized profile preview: a monospace block of the redacted profile JSON, with identifier-category chips above (`account-id`, `ssn`, etc.).
    - Recommendation paragraph: the agent's 2–4 sentence top-level recommendation.
    - Holding-advice table: columns ticker, asset class, current weight (%), suggested weight (%), rationale.
    - Audit section at bottom: audit timestamp and SHA-256 digest (first 8 chars visible, full digest on hover).
- Status pill colours: REQUESTED=muted, SANITIZED=blue, ADVISING=yellow, RESPONSE_RECORDED=blue, AUDITED=green, FAILED=red.
- Risk rating badge colours: CONSERVATIVE=blue, MODERATE=yellow, AGGRESSIVE=red.
- Weight delta indicator: an arrow (↑/↓/=) beside the suggested weight column; red for increases in equity weight when account type is IRA/ROTH_IRA and client tolerance is CONSERVATIVE.

The raw `clientId` and raw market values are never displayed in the UI's holding-advice view — only the sanitized form appears. Advisors who need the raw profile fetch `/api/advisories/{id}` and read `request.profile` from the JSON. This demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw data.
