# UI mockup — akka-stateful-memory-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Hello World Stateful Agent</title>`.

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
- Headline: `Hello World <span class="accent">Stateful Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **New conversation** (or pick a seeded starter from the dropdown).
  3. Send a few messages about yourself — preferences, projects, timezone.
  4. Watch the `human` memory block update after each turn; restart the service and send another message to see the memory survive.
- Card **How it works**: one paragraph on submit → agent call → sanitize → patch; one paragraph on the two governance mechanisms (PII sanitizer on memory writes, periodic drift check).
- Card **Components**: rows per component (ConversationEntity, MemoryWriterConsumer, ConversationWorkflow, MemoryAgent, DriftEvaluator, ConversationView, ConversationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 DriftEvaluator as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, E1). ID badges coloured: S1 green (sanitizer), E1 blue (eval-periodic).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Watch the memory update.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: three-column.
  - **Left column** — conversation list.
    - Button **+ New conversation** at the top. Optional dropdown for seeded starters (Software Engineer, Travel Planner, Cooking Advisor).
    - One card per conversation, newest-first. Each card shows the conversation id (short), age, and the last message preview (truncated to 60 chars).
  - **Centre column** — chat pane.
    - Scrollable message history. User messages right-aligned; agent messages left-aligned. Each message shows its timestamp.
    - Input row at the bottom: text input + **Send** button + optional **Submitted by** field (defaults to `"user"`).
    - A loading indicator appears while the workflow is running.
  - **Right column** — memory blocks pane.
    - **Persona block** card: block id label, last-updated timestamp, content text area (read-only). A diff chip `+N −M` shows lines added/removed on the last turn.
    - **Human block** card: same layout as persona block. On turns where the block was unchanged, the diff chip is absent.
    - **Drift annotation** section below the human block: shows the most recent `DriftAnnotation` if present. Risk score displayed as a colour-coded chip: green (< 0.4), yellow (0.4–0.7), red (≥ 0.7). Shows `reason` and `atTurnNumber`. If no drift eval has fired yet, this section shows `"Evaluates every 10 turns"` in muted text.

- Status indicators: while the workflow is in-flight the chat pane shows a spinner and the send button is disabled. Once `MemoryPatchApplied` arrives over SSE, the right column updates without a page reload.
- Memory block content is shown as plain text. No markdown rendering in the blocks — the content is a short bulleted list written by the agent.
