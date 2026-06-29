# UI mockup — slack-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Slack Assistant</title>`.

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
- Headline: `Slack<span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no credential export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded Slack message from the dropdown or paste a custom message payload.
  3. Click **Send to agent**.
  4. Watch the card transition through SANITIZED → REPLYING → REPLY_RECORDED → AUDITED.
- Card **How it works**: one paragraph on receive → sanitize → reply → audit; one paragraph on the two governance mechanisms (secret sanitizer, reply guardrail).
- Card **Components**: rows per component (MessageEntity, SecretSanitizer, MessageWorkflow, ChannelAssistantAgent, ReplyGuardrail, AuditRecorder, MessageView, MessageEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 AuditRecorder as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `secrets-or-credentials: true` declaration in Data is filled and prominent. `decisions.authority_level = answer-and-escalate` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Read the reply.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded message` (5 options from `messages.jsonl` / custom), `Channel ID` text input, `Channel name` text input, `User ID` text input, `Message text` textarea (pre-filled when a seeded message is selected), and a yellow `Send to agent` button.
    - Live list below: one card per message, newest-first. Each card shows status pill, action badge (when reply landed), guardrail iteration chip (when audit landed), channel name, age.
  - **Right column** — Selected-message detail.
    - Header: status pill + action badge + channel name.
    - Channel context: channel ID, channel name, user ID, received-at timestamp.
    - Sanitized text preview: a monospace block of the scrubbed message window, with secret-category chips above (`slack-token`, `gh-pat`, `aws-key-id`, etc.) — empty chip row if no secrets found.
    - Reply section: action badge + response text. If `runbookUrl` is present, a link chip below the text.
    - Audit section at bottom: a `Candidates / Rejected` count chip (e.g., `3 / 2`) and the outcome string. Rejected count > 0 highlights the chip in yellow.
- Status pill colours: RECEIVED=muted, SANITIZED=blue, REPLYING=yellow, REPLY_RECORDED=blue, AUDITED=green, FAILED=red.
- Action badge colours: ANSWER=green, CLARIFY=blue, RUNBOOK_REF=purple, ESCALATE=yellow.

The raw `triggerText` is never displayed on this screen — only the sanitized form. Operators who need the raw text fetch `/api/messages/{id}` and read `context.triggerText` from the JSON. This is intentional: the UI shows what the model actually received, even though the audit trail keeps the original.
