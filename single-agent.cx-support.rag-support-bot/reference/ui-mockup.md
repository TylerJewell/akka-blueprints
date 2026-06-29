# UI mockup — rag-support-bot

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SupportFlow Lite</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Support<span class="accent">Flow Lite</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded query (Order Status / Returns / Account / Technical) or type your own.
  3. Click **Send**.
  4. Watch the card transition through SANITIZED → RETRIEVING → REPLYING → REPLY_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on receive → sanitize → retrieve → reply → eval; one paragraph on the two governance mechanisms (PII sanitizer, passage-anchoring guardrail) and the on-decision grounding scorer.
- Card **Components**: rows per component (ConversationEntity, KnowledgeBase, MessageSanitizer, ConversationWorkflow, SupportAgent, ReplyGuardrail, GroundingScorer, ConversationView, ConversationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = automated` and `oversight.human_in_loop = false` are the distinctive answers (this system sends replies directly to customers — no human reviews each reply before delivery). Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Get a grounded reply.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Query panel + live list.
    - Query panel: dropdown `Topic filter` (Any / Orders / Returns / Account / Technical), `Your question` textarea with a "Load seeded example" link that fills the textarea with one of the four seed queries, `Submitted by` text input (auto-populated with a session token), and a yellow `Send` button.
    - Live list below: one card per conversation, newest-first. Each card shows status pill, outcome badge (when reply landed), grounding score chip (when eval landed), query preview (first 60 chars of sanitized query), age.
  - **Right column** — Selected-conversation detail.
    - Header: status pill + outcome badge + grounding score chip + query preview.
    - Sanitized query: the redacted query text with PII category chips above (`order-id`, `email`, etc.).
    - Retrieved passages: a collapsible list showing passage id, article title, topic chip, and passage text for each retrieved passage.
    - Reply section: the `answerText` paragraph, a numbered source-citation list (passage id → first 80 chars of passage text), and outcome badge.
    - Grounding eval at bottom: a 1–5 score widget and the one-line rationale. Score ≤ 2 highlights the card border red and shows a "Flag for review" label.
    - Escalation notice: if `escalation == true`, a yellow banner shows `escalationReason`.
- Status pill colours: RECEIVED=muted, SANITIZED=blue, RETRIEVING=yellow, REPLYING=yellow, REPLY_RECORDED=blue, EVALUATED=green, FAILED=red.
- Outcome badge colours: ANSWERED=green, PARTIALLY_ANSWERED=yellow, ESCALATE=orange.
- Grounding score chip: score 4–5=green, score 3=yellow, score 1–2=red.

The raw query is never displayed on this screen — only the sanitized form. Support managers who need the raw text fetch `/api/conversations/{id}` and read `query.rawQuery` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
