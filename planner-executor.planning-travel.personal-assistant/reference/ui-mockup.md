# UI mockup — line-personal-assistant

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: LINE Personal Assistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram on the Architecture tab renders state labels invisible and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `LINE Personal <span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — send a LINE message via the App UI chat, watch the planner route it, expand the conversation card to see the plan and execution result.
- Card **How it works**: one paragraph naming the components, the plan → guard → confirm → execute loop, and the two integration executors (Calendar and Gmail).
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/confirmations/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (3 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text, drive sublabel, and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `hitl` blue (`H1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Chat with the assistant. <span class="accent">Watch the planner decide.</span>` Subtitle: `The simulator drips a message every 90 s so the page is never empty.`
- **Simulated LINE chat pane** (left column): shows a chat bubble interface with the user's message and the assistant's replies. Each reply round-trip corresponds to one conversation. A text input at the bottom lets testers send new messages.
- **Conversation list** (right column): one card per conversation; left border coloured by status (PLANNING = muted, AWAITING_CONFIRMATION = yellow, AWAITING_CLARIFICATION = blue, EXECUTING = indigo, COMPLETED = green, CANCELLED = orange, BLOCKED = red, EXPIRED = pale grey, FAILED = dark red).
  - Header row: original text (first 80 chars), status pill, elapsed time.
  - Click to expand:
    - **Plan**: intent pill (CREATE_EVENT / LIST_EVENTS / SEND_EMAIL / CLARIFY), reasoning text, parameter fields.
    - **Confirmation gate** (only when status is AWAITING_CONFIRMATION or later): recipient, subject, a green `Approve` button and a red `Deny` button wired to `POST /api/confirmations/{id}/reply`. When status is `APPROVED`, show the timestamp. When `DENIED` or `EXPIRED`, show the outcome.
    - **Result** (only when status is COMPLETED): `ExecutionResult.summary` paragraph.
    - **Blocker reason** (when status is BLOCKED): the guardrail rejection message.
