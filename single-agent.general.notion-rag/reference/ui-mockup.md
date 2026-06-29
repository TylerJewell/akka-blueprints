# UI mockup — notion-rag

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: NotionRAG</title>`.

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
- Headline: `Notion<span class="accent">RAG</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded question ("Which products support SSO?") or type your own.
  3. Click **Ask**.
  4. Watch the question transition through ROWS_RETRIEVED → ANSWERING → ANSWERED with source citations.
- Card **How it works**: one paragraph on submit → retrieve → answer; one paragraph on the grounding guardrail.
- Card **Components**: rows per component (SessionEntity, NotionRetriever, QueryWorkflow, NotionQueryAgent, GroundingGuardrail, SessionView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = false` are the distinctive answers for this lookup use-case. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Get grounded answers.</span>`. Subtitle: `One agent, one grounding guardrail, bound to your Notion database.`
- Layout: two-column.
  - **Left column** — Question panel + session list.
    - Question panel at top: `Question` textarea (tall single-line), three seeded-question chips below it ("Which products support SSO?", "Enterprise tier pricing?", "Deprecated items?"), `Session label` text input, `Asked by` text input, and a yellow `Ask` button.
    - Session list below: one row per session, newest-first. Each row shows session label, question count, last-activity age.
  - **Right column** — Active session conversation.
    - Session header: label + status pill + created-at timestamp.
    - Conversation: chronological list of question turns, newest at bottom. Each turn:
      - Question text in a user-bubble.
      - Row-retrieval summary: `{n} rows retrieved` chip + database name.
      - Answer text in an agent-bubble with confidence chip (HIGH=green, MEDIUM=blue, LOW=yellow, NO_DATA=muted).
      - Citation table: columns row ID (truncated), property name, excerpt (italic), match reason. Rows alternate light/dark.
    - Status pill on each turn: SUBMITTED=muted, ROWS_RETRIEVED=blue, ANSWERING=yellow, ANSWERED=green, FAILED=red.
- NO_DATA turns: the agent-bubble background is highlighted amber. The citation table is empty with a "No matching rows were found" note.
- FAILED turns: the agent-bubble background is highlighted red. Shows the error reason.
