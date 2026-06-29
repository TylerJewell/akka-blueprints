# UI mockup — long-term-memory-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Agent with Long-Term Memory</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Memory<span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Enter a userId and click **New Session**.
  3. Type a message and click **Send** — the reply appears within seconds.
  4. Start a second session with the same userId; ask about something from the first session and watch the agent recall it.
- Card **How it works**: one paragraph on message → recall → respond → extract; one paragraph on the PII sanitizer between raw memory output and persistent storage.
- Card **Components**: rows per component (SessionEntity, MemoryEntity, MemorySanitizer, MemoryExtractionWorkflow, MemoryAgent, SessionView, MemoryView, ConversationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 2 Views, 1 Consumer, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `pii_sanitization_stage = post-agent-extraction` is the distinctive answer (contrasting with pre-call sanitizers). Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (S1). ID badge coloured green (sanitizer).
- Row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Talk. <span class="accent">Be remembered.</span>`. Subtitle: `One agent, one governance mechanism, persistent memory across sessions.`
- Layout: two-column.
  - **Left column** — Session rail.
    - New-session form at top: `User ID` text input, `Session label` text input (optional), and a yellow **New Session** button.
    - Session list below: one card per session, newest-first. Each card shows status pill, userId, session label, turn count, age.
    - Clicking a session card opens it in the right column.
  - **Right column** — Active session pane. Divided into two sections:
    - **Conversation section** (top ~65% of column):
      - Session header: status pill + session label + userId.
      - Conversation thread: messages and replies in chat-bubble style (user messages right-aligned, agent replies left-aligned). Each reply shows a timestamp.
      - PENDING turns show a spinner instead of a reply bubble.
      - Message input at bottom: textarea + **Send** button. Disabled when session status is `CLOSED`.
    - **Memory section** (bottom ~35% of column):
      - Header: `Stored memories for {userId}` + entry count chip.
      - Memory list: one row per `MemoryEntry`, newest-first. Each row shows the `kind` chip (FACT / PREFERENCE / RELATIONSHIP / CONTEXT), `sanitizedText`, `relevanceScore` bar, and `recordedAt` age. If `piiCategoriesRedacted` is non-empty, a small lock icon with a tooltip lists the categories.
- Status pill colours: ACTIVE=green, CLOSING=yellow, CLOSED=muted.
- Turn status: PENDING=yellow spinner, REPLIED=none (just shows reply), FAILED=red border on the turn bubble.
- Kind chip colours: FACT=blue, PREFERENCE=purple, RELATIONSHIP=teal, CONTEXT=muted.
