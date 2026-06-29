# UI mockup — mem0-react-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Mem0 ReAct Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Mem0 Re<span class="accent">Act</span> Agent`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Type a question that mentions a personal preference (e.g., "I prefer Fahrenheit. What is 100°C?").
  3. Click **Send** and watch the turn resolve with tool-call steps visible.
  4. Start a **New session** and ask a follow-up question — the agent recalls the preference without being told again.
- Card **How it works**: one paragraph on message → ReAct loop → store fact → sanitize → persist; one paragraph on the two governance mechanisms (PII sanitizer on the memory write path, human-on-the-loop drift monitor).
- Card **Components**: rows per component (SessionEntity, MemoryEntity, FactSanitizer, MemoryWriteWorkflow, DriftMonitor, Mem0ReactAgent, SessionView, MemoryView, AgentEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 2 Views, 2 Consumers, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. The note that `pii_handled_by_sanitizer_before_llm = false` (PII may reach the model in a user message) but `pii_handled_by_sanitizer_before_persist = true` should be clearly explained. `oversight.human_on_loop = true` and `oversight.human_in_loop = false` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, H1). ID badges coloured: S1 green (sanitizer), H1 amber (human-on-the-loop / monitoring).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Build a memory.</span>`. Subtitle: `One agent, facts that persist across sessions.`
- Layout: two-column.
  - **Left column** — Conversation thread.
    - Header bar: `Session` label + `sessionId` chip + current `userId` chip + **New session** button.
    - Conversation thread: alternating user message bubbles (right-aligned, yellow) and agent answer bubbles (left-aligned, dark). Each agent bubble has a collapsible **Tool calls** accordion showing tool name, input, output, and timestamp for each `ToolCall` in the turn.
    - Input row at the bottom: `userId` text input (pre-filled from last session), message textarea, and a yellow **Send** button. Send is disabled while a turn is in flight.
    - Loading indicator: a spinning ellipsis replaces the Send button while the agent is reasoning.
  - **Right column** — Memory panel.
    - Header: `Long-term memory for` + userId chip + fact count badge.
    - Drift monitor strip: a warning banner for each `MemoryDriftSignaled` event, showing userId, fact count, and threshold. Hidden when no signals exist.
    - Facts list: one row per persisted fact, sorted newest-first. Each row shows: `factId` chip (muted), `cleanText`, `piiCategoriesFound` chips (colour-coded: email=red, phone=orange, ssn=red, etc.), `persistedAt` age. Row border colour: PERSISTED=green, FAILED=red, others=muted.
    - A small **Status** sub-header above the facts list with counts: PERSISTED / SANITIZED / SANITIZE_PENDING / FAILED.
- Status pill colours (session): OPEN=muted, ACTIVE=yellow, CLOSED=grey.
- Status pill colours (memory): SANITIZE_PENDING=muted, SANITIZED=blue, PERSISTED=green, FAILED=red.
- The `rawText` field of a memory fact is never displayed in the UI — only `cleanText` is shown. The raw text is accessible via `GET /api/memory/{factId}` in the full JSON response.
