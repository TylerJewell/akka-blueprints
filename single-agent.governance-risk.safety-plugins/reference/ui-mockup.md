# UI mockup — safety-plugins

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SafetyPlugins</title>`.

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
- Headline: `Safety<span class="accent">Plugins</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded safety profile (GENERAL / CHILDREN / ENTERPRISE) and paste the matching seed payload, or load both with one click.
  3. Click **Submit for screening**.
  4. Watch the card transition through SANITIZED → SCREENING → DECISION_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → sanitize → input-guard → screen → output-guard → eval; one paragraph on the four governance mechanisms (PII sanitizer, before-llm-call guardrail, after-llm-response guardrail, on-decision eval).
- Card **Components**: rows per component (SafetyEntity, PayloadSanitizer, SafetyWorkflow, SafetyAgent, InputGuardrail, OutputGuardrail, DecisionEvaluator, SafetyView, SafetyEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 2 Guardrails + 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = blocking` and `oversight.human_in_loop = false` with `oversight.human_on_loop = true` are the distinctive answers for an automated safety filter. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with four rows (S1, G1, G2, E1). ID badges coloured: S1 green (sanitizer), G1 red (before-llm-call guardrail), G2 orange (after-llm-response guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a payload. <span class="accent">See the decision.</span>`. Subtitle: `One agent, four governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Safety profile` (GENERAL / CHILDREN / ENTERPRISE / custom), `Payload title` text input, `Payload direction` radio (user-to-model / model-to-user), `Payload` textarea (with a "Load seeded example" link that fills profile, title, and body), `Submitted by` text input, and a yellow `Submit for screening` button.
    - Live list below: one card per screening, newest-first. Each card shows status pill, overall action badge (when decision landed), eval score chip (when eval landed), payload title, age. FAILED cards show the failure reason (e.g., `inputGuardrail.hardBlock: injection-pattern-3`).
  - **Right column** — Selected-screening detail.
    - Header: status pill + overall action badge + eval score chip + `payloadTitle`.
    - Submitted safety rules: a small list with `ruleId`, `category` chip, `actionFloor` chip, and the rule description.
    - Sanitized payload preview: a monospace block of the redacted payload, with PII category chips above (`email`, `ssn`, `phone`, etc.).
    - Decision summary: the agent's 1–3-sentence paragraph.
    - Findings table: columns rule id, category chip, action (coloured chip), evidence quote (italic, empty if ALLOW), rationale.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, SCREENING=yellow, DECISION_RECORDED=blue, EVALUATED=green, FAILED=red.
- Overall action badge colours: ALLOW=green, REDACT=yellow, BLOCK=red.
- Finding action chip colours: ALLOW=muted, REDACT=yellow, BLOCK=red.
- Category chip colours: HATE_SPEECH=red, SELF_HARM=red, PROMPT_INJECTION=orange, DATA_EXFILTRATION=orange, SEXUAL_CONTENT=red, VIOLENCE=orange, HARASSMENT=orange, ILLEGAL_GOODS=orange, MISINFORMATION=blue, OTHER=muted.

The raw payload is never displayed on this screen — only the sanitized form. Operators who need the raw text fetch `/api/screenings/{id}` and read `request.rawPayload` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
