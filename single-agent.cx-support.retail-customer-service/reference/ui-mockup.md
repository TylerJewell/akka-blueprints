# UI mockup — retail-customer-service

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: RetailCS</title>`.

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
- Headline: `Retail<span class="accent">CS</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded customer scenario from the dropdown (product inquiry / address change / cancellation attempt).
  3. Type a message or click **Load scenario** to pre-fill.
  4. Click **Send** and watch the conversation thread update with the agent's reply and any order-change chip.
- Card **How it works**: one paragraph on session start → agent invocation → guardrail checks → reply delivery; one paragraph on the two governance mechanisms (before-tool-call state gate, before-agent-response policy check).
- Card **Components**: rows per component (SessionEntity, OrderEntity, SessionWorkflow, CustomerServiceAgent, OrderModificationGuardrail, ReplyPolicyGuardrail, SessionView, OrderView, CustomerServiceEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machines, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 2 Views, 2 HttpEndpoints, plus 2 Guardrail classes).
- Four mermaid cards:
  1. Component graph (from PLAN.md)
  2. Interaction sequence — J2 happy path
  3. State machine — SessionEntity
  4. State machine — OrderEntity
  All cards render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `decisions_surface: action-with-guardrail` and `human_in_loop: false` / `human_on_loop: true` are the distinctive answers for this pattern. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, G2). Both are guardrail type; G1 badge coloured amber (before-tool-call), G2 badge coloured red (before-agent-response).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Change an order.</span>`. Subtitle: `One agent, two guardrails at every boundary.`
- Layout: two-column.
  - **Left column** — Compose panel + session list.
    - Compose panel: dropdown `Scenario` (Product inquiry / Address change / Order cancellation / Custom), `Customer ID` text input, `Order ID` text input (optional), `Message` textarea, and a green `Send` button. A `Load scenario` link fills the message and order ID from the selected seeded scenario.
    - Session list below: one card per session, newest-first. Each card shows status pill, last-turn indicator (replied / blocked / failed), customer ID, turn count, age.
  - **Right column** — Selected-session conversation thread.
    - Header: session status pill + customer ID + session ID + age.
    - Conversation thread: alternating customer messages (right-aligned, light background) and agent replies (left-aligned, dark background), chronological. Each agent reply card includes:
      - Reply text.
      - If `orderChange` is present: an **Order modified** chip with the change type and order ID.
      - If `status == GUARDRAIL_BLOCKED`: a **Guardrail blocked** chip with the rejection code.
    - Empty state: "Select a session from the list or start a new conversation."
- Status pill colours: OPEN=muted, ACTIVE=green, CLOSED=grey.
- Turn status indicators: REPLIED=green dot, GUARDRAIL_BLOCKED=amber dot, FAILED=red dot.
- Order-change chip: teal background, order ID + change type.
- Guardrail-block chip: amber background, policy code.
