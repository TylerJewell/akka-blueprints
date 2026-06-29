# UI mockup — personal-finance-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Personal Finance Assistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Personal Finance <span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded account set (individual-checking / joint-household / small-business).
  3. Type a question (e.g., "What did I spend on dining last month?" or "Transfer $100 to savings").
  4. Watch the card transition through SANITIZED → ANSWERING → ANSWERED; inspect the tool trace.
- Card **How it works**: one paragraph on submit → sanitize → agent → answer; one paragraph on the two governance mechanisms (PII sanitizer, before-tool-call write guardrail).
- Card **Components**: rows per component (QueryEntity, TransactionSanitizer, QueryWorkflow, FinanceAssistantAgent, FinanceTools, WriteGuardrail, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, 1 AgentTool provider, 1 Guardrail).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` and `payment-card-data: true` declarations are filled and prominent. `decisions.authority_level = act-on-behalf` and `oversight.human_in_loop = false` are the distinctive answers for this domain — the agent acts without a separate human approver; the guardrail is the control. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Get an answer.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Account set` (individual-checking / joint-household / small-business), `Query` textarea (placeholder: "How much did I spend on groceries last month?"), `Principal` text input (defaults to "user-0001"), and a yellow `Ask` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, age, and first 60 characters of query text.
  - **Right column** — Selected-query detail.
    - Header: status pill + `queryText`.
    - Sanitized context block: account list with tokenised IDs, PII-token count badge (e.g., "6 PII tokens replaced").
    - Agent answer: the natural-language paragraph.
    - Structured data block: if present, rendered as a table (category / total / count for spending summaries; account / available / currency for balance queries).
    - Tool-call trace: collapsed by default; expandable. Each row shows tool name, outcome badge (NOT_A_WRITE / APPROVED / BLOCKED / ERROR), and a timestamp. Clicking a row shows the arguments and result JSON.
    - Write-outcome callout: if any tool call has `outcome != NOT_A_WRITE`, a prominent callout appears above the trace. APPROVED = green, BLOCKED = red with the block reason, ERROR = orange.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, ANSWERING=yellow, ANSWERED=green, FAILED=red.
- Write-outcome badge colours: NOT_A_WRITE=muted, APPROVED=green, BLOCKED=red, ERROR=orange.

Raw transaction data (`request.rawTransactions`) is never shown in the UI — only the sanitized form. Users who need the raw records fetch `/api/queries/{id}` directly. This is intentional: the UI demonstrates that the model's input is the tokenised form even though the audit trail keeps the originals.
