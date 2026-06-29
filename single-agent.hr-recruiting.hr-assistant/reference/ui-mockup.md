# UI mockup — hr-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: HR Assistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `HR<span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded query (PTO inquiry / parental leave / job posting lookup) or type your own.
  3. Optionally enter an Employee ID to include profile context.
  4. Click **Submit query** and watch the card transition through SANITIZED → ANSWERING → ANSWER_RECORDED.
- Card **How it works**: one paragraph on submit → sanitize (PII + special-category) → answer → record; one paragraph on the three governance mechanisms (PII sanitizer, special-category sanitizer, before-agent-response guardrail).
- Card **Components**: rows per component (QueryEntity, QuerySanitizer, QueryWorkflow, HrPolicyAgent, PolicyAnswerGuardrail, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` and `special_category: true` declarations in Data are both filled and prominent. `decisions.authority_level = assist-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, S2, G1). ID badges coloured: S1 green (sanitizer / pii), S2 amber (sanitizer / special-category), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask an HR question. <span class="accent">Get a cited answer.</span>`. Subtitle: `One agent, two sanitizer passes, one output guardrail.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded query` (PTO inquiry / Parental leave / Job posting search / custom), `Query text` textarea (pre-filled when a seeded query is selected), `Employee ID` text input (optional, labelled "optional"), and a yellow `Submit query` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, applicability badge (when answer landed), query text excerpt (first 60 chars), age.
  - **Right column** — Selected-query detail.
    - Header: status pill + applicability badge + query text.
    - Sanitization summary: two chip groups — PII categories found (blue chips) and special-category fields found (amber chips). If both lists are empty, show a green "No sensitive fields detected" indicator.
    - Employee profile chip: department, title, hire date, work location — rendered as a compact info block. Hidden if no employeeId was supplied.
    - Answer text: the agent's 2–5 sentence paragraph.
    - Citations table: columns policy id, section reference, quoted passage (italic).
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, ANSWERING=yellow, ANSWER_RECORDED=green, FAILED=red.
- Applicability badge colours: APPLICABLE=green, PARTIALLY_APPLICABLE=yellow, NOT_APPLICABLE=muted.
- PII chip colour: blue. Special-category chip colour: amber.

The raw query text is never displayed on this screen — only the sanitized form. An HR professional who needs the raw text fetches `/api/queries/{id}` and reads `request.rawQueryText` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
