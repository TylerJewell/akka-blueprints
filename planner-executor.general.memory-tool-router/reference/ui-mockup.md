# UI mockup — memory-tool-router

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: AI Chatbot with Long-Term Memory and Dynamic Tool Routing</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram on the Architecture tab renders state names invisible and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `AI Chatbot with Long-Term Memory and <span class="accent">Dynamic Tool Routing</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — send a message in the App UI tab, watch the router classify and dispatch, expand the row to see the turn log and memory store.
- Card **How it works**: one paragraph naming the components, the turn loop steps, and the four tool executors.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/conversations/{id}/memory` and `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (6 agents, 1 workflow, 4 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text and widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".
- Note: `data.data_classes.pii = true` is answered; the PII row renders at full opacity with the chip selected.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `sanitizer` green (`S1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Watch memory grow.</span>` Subtitle: `The simulator drips a turn every 90 s so the page is never empty.`
- Form card: session ID field (auto-populated with `s-demo`), message text field, `Send` button (yellow).
- **Operator controls pane** (top right of the App UI tab):
  - If `halted=false`: yellow `Halt new dispatches` button, a free-text reason field.
  - If `halted=true`: muted `HALTED` pill with the reason and timestamp, plus a `Resume` button.
  - The pane reflects every `control-update` SSE event live.
- Live list: cards per conversation; left border coloured by status (IDLE = muted, PROCESSING = blue, HALTED = orange, STUCK = pale red, ENDED = green).
  - Header row: session ID, last message text (first 60 chars), status pill, turn count, elapsed time.
  - Click to expand:
    - **Message history**: alternating user/assistant message bubbles (user on right, assistant on left), most recent last.
    - **Turn log**: vertical timeline of `TurnEntry` rows. Each row shows turn number, tool-used pill (`CALCULATOR` blue, `KNOWLEDGE_BASE` yellow, `WEB_LOOKUP` cyan, `CODE_RUNNER` purple, `NONE` muted), verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red), reply text (collapsed `<pre>`, click to expand), and blocker reason if any.
    - **Memory store**: a table of the session's `MemoryItem` entries — key, value (redacted spans rendered in italics with a tooltip showing the redaction tag), kind pill, recorded time. Refreshes on each SSE update. If the store is empty, shows "No memories yet for this session."
    - **Halt reason** (when applicable): one-paragraph block coloured orange.
