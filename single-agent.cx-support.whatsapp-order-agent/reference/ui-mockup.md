# UI mockup — whatsapp-order-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: WhatsApp Order Agent</title>`.

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
- Headline: `WhatsApp Order <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded session from the dropdown (or start a new one with a customer ID).
  3. Type a customer message (e.g., "Order 2 Blue Widgets") and click **Send**.
  4. Watch the session transition through ACTIVE → COMPLETING, with the tool-calls panel showing the guardrail decision.
- Card **How it works**: one paragraph on session-start → agent turn → sanitize → score; one paragraph on the three governance mechanisms (before-tool-call guardrail, PII sanitizer, HITL gate).
- Card **Components**: rows per component (SessionEntity, OrderEntity, ConversationSanitizer, OrderWorkflow, OrderAgent, ToolGuardrail, TurnScorer, SessionView, OrderEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous-write` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and displayed in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, S1, H1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer), H1 yellow (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Place an order.</span>`. Subtitle: `One agent, three governance controls around every tool call.`
- Layout: two-column.
  - **Left column** — Session list + HITL queue.
    - **Session controls**: customer ID text input, WhatsApp phone number text input, and a yellow `Start session` button. Or pick from a seeded sessions dropdown.
    - **Session list** below: one card per session, newest-first. Each card shows status pill, customer ID, last agent reply snippet, age.
    - **HITL queue** section (visible when at least one session is in `AWAITING_APPROVAL`): one card per pending order showing order total, product summary, and **Approve** / **Reject** buttons.
  - **Right column** — Selected session's chat thread.
    - Header: status pill + customer ID + session age.
    - Chat thread: alternating customer-message bubbles (left-aligned, grey) and agent-reply bubbles (right-aligned, accent). Each agent-reply bubble has:
      - A collapsible **Tool calls** panel listing `toolName`, `outcome` chip (allowed=green / blocked=red), and `blockReason` when blocked.
      - A **PII** chip row (e.g., `phone`, `address`) when `piiCategoriesRedacted` is non-empty.
      - A turn eval score badge (1–5) with a tooltip showing `TurnEval.rationale`.
    - Message input: text input + yellow `Send` button.
- Status pill colours: IDLE=muted, ACTIVE=yellow, AWAITING_APPROVAL=orange, COMPLETING=blue, CLOSED=green, FAILED=red.
- Tool outcome chip colours: allowed=green, blocked=red.
- Turn eval score badge colours: 5=green, 4=blue, 3=yellow, 2=orange, 1=red.

The raw customer message text (pre-sanitization) is never displayed in the chat thread — only the sanitized form. Operators who need the raw text fetch `/api/sessions/{id}` from the API and read the `TurnReceived` event payload. This is intentional: the UI demonstrates that the durable view contains only redacted data.
