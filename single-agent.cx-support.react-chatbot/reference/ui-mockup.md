# UI mockup — react-chatbot

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Basic Tool-Calling Chatbot</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Basic Tool-Calling <span class="accent">Chatbot</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **New conversation** and type a question (or use the pre-loaded seed: "Where is my order ORD-003?").
  3. Click **Send**.
  4. Watch the turn transition through PROCESSING → COMPLETED and the reply bubble appear with the tool-call trace.
- Card **How it works**: one paragraph on message → agent tool loop → reply; one paragraph on the `before-agent-response` guardrail and the content-policy check.
- Card **Components**: rows per component (ConversationEntity, ConversationWorkflow, ChatAgent, ReplyGuardrail, ToolRegistry, ConversationView, ChatEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machines, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 1 ToolRegistry as supporting classes).
- Four mermaid cards (component graph, sequence, turn state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `inform-only` authority level and `human_in_loop = false` are the distinctive answers. Many deployer-specific fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Get an answer.</span>`. Subtitle: `One agent, one guardrail, three in-process tools.`
- Layout: two-column.
  - **Left column** — Conversation list + new-conversation button.
    - **New conversation** button at the top. Clicking it opens an inline form: `Title` text input + `Your ID` text input + **Start** button.
    - Below: live list of conversations, newest-first. Each entry shows conversation title, last-turn status pill, and relative age. Clicking a conversation selects it in the right pane.
  - **Right column** — Selected-conversation turn history + message input.
    - Header: conversation title + status pill.
    - Turn history: a scrollable thread. Each turn renders as:
      - A right-aligned user bubble with the message content and a timestamp.
      - A collapsible **Reasoning** row (collapsed by default) showing the tool-call trace: for each `ToolCall`, a chip with `toolName`, a one-line `inputSummary`, and a one-line `resultSummary`.
      - A left-aligned agent reply bubble with the `content` text and a timestamp. If the turn is `PROCESSING`, a pulsing placeholder replaces the reply bubble.
      - If the turn is `FAILED`, the reply bubble slot shows a red error card with the `failureReason`.
    - Message input at the bottom: `Message` textarea + `Your ID` text input + **Send** button. The Send button is disabled if the conversation has a `PROCESSING` turn. Attempting to send in that state shows an inline "Wait for the current reply" tooltip.
- Status pill colours: PROCESSING=yellow, COMPLETED=green, FAILED=red.
- Conversation status pill: ACTIVE=green, CLOSED=muted.

Tool-call trace is collapsed by default because most users care about the answer, not the reasoning steps. Developers and auditors expand it to verify which tools were called and what they returned.
