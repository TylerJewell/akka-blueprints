# UI mockup — rag-baseline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: RAG Sample</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `RAG <span class="accent">Sample</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded question from the dropdown or type your own.
  3. Click **Ask**.
  4. Watch the card transition through PASSAGES_RETRIEVED → ANSWERING → ANSWER_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → retrieve → answer → eval; one paragraph on the governance mechanism (on-decision groundedness eval).
- Card **Components**: rows per component (QueryEntity, PassageRetriever, VectorStore, QueryWorkflow, AnswerAgent, GroundednessEvaluator, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 plain-class Evaluator + 1 plain-class VectorStore as supporting components).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. `decisions.authority_level = informational-only` is the distinctive answer. Most deployer-specific fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (E1). ID badge coloured blue (eval-event).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the grounded answer.</span>`. Subtitle: `One agent, retrieved passages as attachments, deterministic groundedness eval.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded questions` (with 3–5 pre-written options and a "Custom" entry that enables free text input), `Question` textarea (pre-filled when a seeded option is selected), `Submitted by` text input, and a yellow `Ask` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, groundedness chip (when eval landed), first 60 chars of the question text, age.
  - **Right column** — Selected-query detail.
    - Header: status pill + groundedness chip + question text.
    - Retrieved passages section: a scrollable list of passage cards, each showing `passageId`, `documentTitle`, similarity score bar, and the passage text.
    - Answer prose: the agent's 1–4 sentence paragraph.
    - Citation table: columns passage id, passage excerpt (italic), claim supported.
    - Groundedness section at bottom: a 1–5 score chip (coloured green ≥ 4, yellow = 3, red ≤ 2), the one-line rationale, and the `usedPassages` count.
- Status pill colours: SUBMITTED=muted, PASSAGES_RETRIEVED=blue, ANSWERING=yellow, ANSWER_RECORDED=blue, EVALUATED=green, FAILED=red.
- Groundedness chip colours: 5=bright-green, 4=green, 3=yellow, 2=orange, 1=red.
- Cards with groundedness ≤ 2 have a red left border to draw the user's eye.
- The retrieved passages list is always visible in the detail pane once the query reaches `PASSAGES_RETRIEVED`, so the user can compare the answer to its evidence source directly.
