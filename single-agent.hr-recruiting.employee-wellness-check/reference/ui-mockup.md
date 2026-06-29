# UI mockup — wellness-check-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Wellness Check Agent</title>`.

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
- Headline: `Wellness <span class="accent">Check Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Launch a seeded campaign (Morale Pulse / Burnout Assessment / Return-to-Office Survey).
  3. Submit a simulated employee response using a seed example.
  4. Watch the card transition through SANITIZED → ANALYSING → ANALYSIS_RECORDED → EVALUATED (or ESCALATED for crisis responses).
- Card **How it works**: one paragraph on submit → sanitize → analyse → surveillance; one paragraph on the three governance mechanisms (special-category sanitizer, guardrail with crisis intercept, post-market-surveillance human-on-the-loop).
- Card **Components**: rows per component (`CampaignEntity`, `CheckInEntity`, `ResponseSanitizer`, `CheckInWorkflow`, `WellnessCheckAgent`, `ResponseGuardrail`, `MoraleSurveillance`, `CheckInView`, `CampaignView`, `CheckInEndpoint`, `AppEndpoint`) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 2 Views, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Surveillance evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `special_category_wellness: true` declaration in Data is filled and prominent. `emotion_recognition_in_workplace: true` (narrowed to voluntary text classification) is highlighted with a note that `emotion_recognition_scope` must be documented. `decisions.authority_level = recommend-only`, `oversight.human_on_loop = true`, and `crisis_routing_to_human: true` are the distinctive answers. Deployer-specific fields are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, H1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), H1 purple (hotl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Launch a campaign. <span class="accent">Read the morale signal.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Campaign + check-in panel.
    - Campaign panel at top: `Campaign` dropdown seeded with three options (Morale Pulse 5Q / Burnout Assessment 4Q / Return-to-Office 3Q / custom), `Campaign name` text input, `Audience label` text input, `Scheduled at` datetime input (defaults to now), `Created by` text input, and a yellow `Launch campaign` button.
    - Check-in submission panel below: `Campaign` dropdown (lists active campaigns), `Employee ref` text input, per-question answer inputs (rendered dynamically from the selected campaign's questions), and a `Submit response` button plus a "Load seeded example" link.
    - Live check-in list below: one card per check-in, newest-first. Each card shows status pill, morale badge (when analysis landed), crisis banner if escalated, document title, age.
  - **Right column** — Selected check-in detail.
    - Header: status pill + morale badge + `employeeRef` pseudonym.
    - Questions sent: a small list of the check-in questions with `questionId` and `kind` chip.
    - Sanitized response: a monospace block showing the redacted answer map, with special-category chip list above (`mental-health-marker`, `disability-marker`, etc.). If no markers were found, shows a green "No special-category markers detected" notice.
    - Analysis summary: morale-level badge (large), the agent's 1–3-sentence interpretation paragraph, and the recommendation.
    - Crisis banner: if `status == ESCALATED`, a full-width red banner reading "Crisis signal detected — routed to human. This check-in is excluded from aggregate morale scores."
    - Surveillance section: `riskFlagRaised` indicator and the one-line rationale. If the campaign's risk flag is raised, shows a yellow **Risk review required** notice linking to the campaign card.
- Status pill colours: RECEIVED=muted, SANITIZED=blue, ANALYSING=yellow, ANALYSIS_RECORDED=blue, ESCALATED=red, EVALUATED=green, FAILED=red.
- Morale badge colours: HIGH=green, MODERATE=blue, LOW=yellow, CRISIS=red.

Raw response answers are never displayed in the right pane — only the sanitized form. Reviewers who need the raw text fetch `GET /api/check-ins/{id}` and read `response.answers` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw text.
