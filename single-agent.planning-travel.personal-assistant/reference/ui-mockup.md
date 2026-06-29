# UI mockup — personal-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: PersonalAssistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Personal<span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded context profile (Travel week / Day-trip / Minimal) and load the matching example, or paste your own.
  3. Type a natural-language request (e.g. "Mark the taxi booking task as done") and click **Send request**.
  4. Watch the card transition through SANITIZED → ACTING → ACTION_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → sanitize → act → eval; one paragraph on the two governance mechanisms (write guardrail, PII sanitizer).
- Card **Components**: rows per component (AssistantEntity, ContextSanitizer, AssistantWorkflow, AssistantAgent, WriteGuardrail, ActionScorer, AssistantView, AssistantEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` and `location-fine-grained: true` declarations in Data are filled and prominent. `decisions.authority_level = execute` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a request. <span class="accent">See the action.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Request panel + live list.
    - Request panel: dropdown `Context profile` (Travel week / Day-trip / Minimal / custom), a `Request` textarea (with a "Load example" link that fills the context dropdown and the textarea), `Submitted by` text input, and a yellow `Send request` button.
    - Live list below: one card per request, newest-first. Each card shows status pill, action type badge (when action landed), eval score chip (when eval landed), request excerpt (first 60 chars), age.
  - **Right column** — Selected-request detail.
    - Header: status pill + action type badge + eval score chip + request excerpt.
    - Original request: the full natural-language request text.
    - Sanitized context preview: a monospace JSON block of the redacted context snapshot, with PII category chips above (`email`, `phone`, `address`, `person-name`, etc.).
    - Action explanation: the agent's 1–2-sentence paragraph.
    - Change set table: columns field, old value (greyed), new value.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, ACTING=yellow, ACTION_RECORDED=blue, EVALUATED=green, FAILED=red.
- Action type badge colours: CREATE_EVENT=teal, CREATE_TASK=teal, UPDATE_TASK=blue, COMPLETE_TASK=green, SET_REMINDER=purple, NO_ACTION=muted.
- Eval score colours: 4–5=green, 3=blue, 1–2=red.

The raw context snapshot is never displayed on this screen — only the sanitized form. Users who need the raw context fetch `/api/requests/{id}` and read `request.rawContext` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
