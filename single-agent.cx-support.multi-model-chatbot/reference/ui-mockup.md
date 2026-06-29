# UI mockup — multi-model-chatbot

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: MultiModelChatbot</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Multi<span class="accent">Model</span>Chatbot`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **New session** and enter a display name.
  3. Type a support question and click **Send**.
  4. Watch the message transition through SANITIZED → REPLIED; observe the PII chips and provider badge.
- Card **How it works**: one paragraph on receive → sanitize → reply; one paragraph on the two governance mechanisms (PII sanitizer, content-moderation guardrail).
- Card **Components**: rows per component (ConversationEntity, MessageSanitizer, ChatWorkflow, ChatAgent, ReplyGuardrail, ContentPolicyChecker, ConversationView, ChatEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 PolicyChecker as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous-action` and `oversight.human_on_loop = true` (supervisor monitoring) are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Get a reply.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Session list.
    - **New session** button at top (opens a small inline form: display name + optional provider dropdown).
    - Session list below: one card per session, newest-first. Each card shows display name, `activeProvider` badge, last-activity age, and a snippet of the most recent turn (REPLIED or FAILED).
  - **Right column** — Selected session chat thread.
    - Session header: display name, `activeProvider` chip (clickable to change provider), session status badge.
    - Chat thread: message bubbles in chronological order. User turns show the sanitized text with PII category chips below (e.g., `email`, `phone`). Agent reply bubbles show `replyText` with a small provider-attribution label (`claude-sonnet-4-6 · anthropic`) and the `repliedAt` timestamp.
    - A `FAILED` turn shows a red error bubble with the failure reason.
    - A `SANITIZED` (awaiting reply) turn shows a spinner bubble.
    - Message input at the bottom: a textarea + **Send** button. Disabled while the session has an in-flight turn (status `RECEIVED` or `SANITIZED`).
- Turn status badge colours: RECEIVED=muted, SANITIZED=blue, REPLIED=green, FAILED=red.
- Provider badge colours: anthropic=purple, openai=teal, googleai-gemini=orange.

The raw user text is never displayed in the chat thread — only the sanitized form. Users and supervisors who need the raw text can fetch `GET /api/chat/{sessionId}` and read each turn's `userText` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the entity preserves the originals for audit.
