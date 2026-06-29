# UI mockup — guardrails-baseline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Guardrails Baseline</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Guardrails <span class="accent">Baseline</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded policy set (finance / healthcare / support) and paste the matching seed message, or load both with one click.
  3. Click **Submit for moderation**.
  4. Watch the card transition through SANITIZED → MODERATING → DECISION_RECORDED → AUDITED.
- Card **How it works**: one paragraph on submit → sanitize → moderate → audit; one paragraph on the three governance mechanisms (PII sanitizer, input guardrail, output guardrail).
- Card **Components**: rows per component (ModerationEntity, MessageSanitizer, ModerationWorkflow, ModerationAgent, InputGuardrail, OutputGuardrail, AuditScorer, ModerationView, ModerationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 2 Guardrails + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = automated-with-human-review` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, G2). ID badges coloured: S1 green (sanitizer), G1 red (guardrail — input), G2 red (guardrail — output).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a message. <span class="accent">Read the decision.</span>`. Subtitle: `One agent, three governance controls around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Policy set` (finance / healthcare / support / custom), `Message title` text input, `Message` textarea (with a "Load seeded example" link that fills both title and body), `Submitted by` text input, and a yellow `Submit for moderation` button.
    - Live list below: one card per moderation, newest-first. Each card shows status pill, verdict badge (when decision landed), audit score chip (when audit landed), message title, age.
  - **Right column** — Selected-moderation detail.
    - Header: status pill + verdict badge + audit score chip + `messageTitle`.
    - Submitted rules: a small list with `ruleId`, `category` chip, and the rule description.
    - Sanitized message preview: a monospace block of the redacted message, with PII category chips above (`email`, `phone`, etc.). Empty chip list when no PII was found.
    - Decision rationale: the agent's 1–3-sentence paragraph.
    - Rule results table: columns rule id, action (coloured badge), confidence (progress bar), explanation.
    - Audit section at bottom: a 1–5 star widget and the one-line note. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, MODERATING=yellow, DECISION_RECORDED=blue, AUDITED=green, FAILED=red.
- Verdict badge colours: ALLOW=green, ESCALATE=yellow, BLOCK=red.
- Action badge colours in rule results: ALLOW=green, ESCALATE=yellow, BLOCK=red.

The raw message is never displayed on this screen — only the sanitized form. Operators who need the raw text fetch `/api/moderations/{id}` and read `request.rawMessage` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
