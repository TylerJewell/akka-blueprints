# UI mockup — text-to-sql-guarded

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Advanced Text-to-SQL Workflow</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Advanced Text-to-SQL <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded questions (or type your own) and click **Run query**.
  3. Watch the card transition through PARSING → PARSED → QUERYING → QUERIED → FORMATTING → FORMATTED (or HALTED if a destructive SQL pattern was generated).
  4. Inspect the rejection-log strip on the card if any phase-gate rejections fired, or the halt reason if the safety halt triggered.
- Card **How it works**: one paragraph on the three task phases (PARSE → QUERY → FORMAT) and the typed handoff between them; one paragraph on the three governance mechanisms (phase-gate guardrail, automatic safety halt, PII sanitizer).
- Card **Components**: rows per component (QueryEntity, QueryPipelineWorkflow, SqlAgent, ParseTools, QueryTools, FormatTools, SqlGuardrail, SqlSafetyInspector, PiiSanitizer, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, 1 SafetyInspector, and 1 Sanitizer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled (employee names, SSNs, account numbers may appear in query results). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, S1). ID badges coloured: G1 red (guardrail), H1 orange (halt), S1 purple (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a financial question. <span class="accent">Get a safe, clean report.</span>`. Subtitle: `One agent, three task phases, destructive SQL blocked, PII redacted.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Question` (with a "Pick a seeded question" dropdown that fills it), and a yellow `Run query` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, question preview (truncated at 60 chars), age, a small orange flame icon if the safety halt fired on this query, and a small red dot if any guardrail rejection fired.
  - **Right column** — Selected-query detail.
    - Header: status pill + question text.
    - Phase panel 1 (SQL): a code block showing the generated SQL. A yellow warning banner replaces it if the query is in `HALTED` status, showing the halt reason and matched pattern. Visible once `parsedQuery` is present.
    - Phase panel 2 (Raw query metadata): column list and row count from the QUERY phase. Note: raw rows are not displayed; only the sanitized report is shown. Visible once `rawResult` is present.
    - Phase panel 3 (Report): narrative paragraph, redaction summary chip (e.g., "3 fields redacted"), and the formatted results table with sanitized values. Visible once `report` is present.
    - Rejection-log strip (only visible if the query has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, PARSING=blue, PARSED=blue, QUERYING=yellow, QUERIED=yellow, FORMATTING=blue, FORMATTED=green, HALTED=orange, FAILED=red.
- The redaction summary chip uses a purple accent to signal PII handling; `redactionCount = 0` shows a grey chip labelled "No PII detected".

Each phase panel renders only when its data is present on the row record. A query in `QUERYING` shows panel 1 (SQL) and a panel 2 spinner. This is the visual proof that the typed handoff between phases is the only path information travels — and that the sanitizer ran before any formatted output appears.
