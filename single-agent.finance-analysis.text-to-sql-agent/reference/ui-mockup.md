# UI mockup — text-to-sql-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Text-to-SQL Agent</title>`.

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
- Headline: `Text-to-<span class="accent">SQL Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded question template or type your own.
  3. Click **Ask**.
  4. Watch the card transition through GENERATING → EXECUTING → EXECUTED → SANITIZED.
- Card **How it works**: one paragraph on question → generate SQL → guardrail → execute → sanitize; one paragraph on the two governance mechanisms (SQL safety guardrail, PII sanitizer).
- Card **Components**: rows per component (QueryEntity, ResultSanitizer, QueryWorkflow, SqlGeneratorAgent, SqlGuardrail, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `failure_modes` list (sql-injection-via-llm, hallucinated-row, pii-leakage-in-result) are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the answer.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Question panel + live list.
    - Question panel: dropdown `Seeded questions` (6 options or "Custom"), a `Question` textarea (pre-filled from dropdown selection), a `Submitted by` text input, and a yellow `Ask` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, question text (truncated), and age.
  - **Right column** — Selected-query detail.
    - Header: status pill + `question` text.
    - Generated SQL block: syntax-highlighted `<pre><code>` block showing the SELECT statement. A small badge shows "Guardrail: passed" (green) or "Guardrail: retried N×" (yellow) if the guardrail fired.
    - SQL explanation: one sentence below the code block in muted text.
    - Result table: column headers from the first row's keys, rows from `sanitized.rows`. `[REDACTED-NAME]` and `[REDACTED-EMAIL]` tokens rendered with a muted badge. PII category chips above the table (`customer-name`, `email`, etc.) when `piiCategoriesFound` is non-empty.
    - Prose summary: the agent's one-sentence answer in a highlighted callout box.
    - If status is `FAILED`, show the failure reason in a red callout instead.
- Status pill colours: SUBMITTED=muted, GENERATING=yellow, EXECUTING=yellow, EXECUTED=blue, SANITIZED=green, FAILED=red.

The raw result rows (containing un-redacted customer data) are never displayed on this screen — only the sanitized form. Analysts who need the raw data fetch `/api/queries/{id}` and read `rawResult.rows` from the JSON directly. This is intentional: the UI demonstrates that the user-facing surface shows only redacted output, even though the audit trail retains the original.
