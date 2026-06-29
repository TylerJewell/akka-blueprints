# UI mockup — restaurant-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Restaurant Assistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Restaurant <span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Start a new chat session or load a seeded example conversation.
  3. Ask about the menu, request a table, or place an order.
  4. Watch the session card transition through ACTIVE → RESERVATION_HELD or ORDER_PLACED in real time.
- Card **How it works**: one paragraph on customer message → agent turn → guardrail check → reply or write commit; one paragraph on the two governance mechanisms (before-agent-response and before-tool-call guardrails).
- Card **Components**: rows per component (SessionEntity, SessionWorkflow, RestaurantAssistantAgent, ResponseGuardrail, ToolCallGuardrail, SessionView, SessionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 2 Guardrail classes + 1 MenuCatalog as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = act-on-behalf` and `oversight.human_on_loop = true` are the distinctive answers for a customer-service agent that takes writes without a human review step per transaction. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, G2). ID badges coloured: G1 red (before-agent-response guardrail), G2 red (before-tool-call guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Chat with the restaurant. <span class="accent">Book a table or order.</span>`. Subtitle: `One agent, two guardrails — one protecting what the customer hears, one protecting what gets written.`
- Layout: two-column.
  - **Left column** — Session list panel.
    - New session button at the top. Optional guest name + phone inputs.
    - Live session list below: one card per session, newest-first. Each card shows status pill, most-recent message excerpt (truncated to 60 chars), age. Clicking a card selects it in the right pane.
    - A "Load seeded example" link populates the right pane with a pre-canned transcript (menu query / reservation / order examples selectable via a small dropdown).
  - **Right column** — Chat thread + input bar.
    - Top: session header showing status pill, guest name, reservation chip (if RESERVATION_HELD), order chip (if ORDER_PLACED).
    - Chat thread: alternating CUSTOMER (right-aligned, accent fill) and ASSISTANT (left-aligned, muted fill) message bubbles. When a reservation commits, an inline confirmation card appears in the thread (table, date, time, party size). When an order commits, an inline order summary card appears (line items, total).
    - Bottom-anchored input bar: a text input and a `Send` button. Disabled while the agent is processing (spinner on the Send button).
- Status pill colours: OPEN=muted, ACTIVE=yellow, RESERVATION_HELD=blue, ORDER_PLACED=green, CLOSED=muted-dark, FAILED=red.
- Guardrail rejection events are not shown to the customer. The UI only displays the agent's final accepted reply.
