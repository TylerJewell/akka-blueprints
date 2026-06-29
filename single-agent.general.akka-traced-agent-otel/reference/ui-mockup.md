# UI mockup — traced-agent-otel

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: TracedAgent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index (Lesson 26):

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Traced<span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded prompt (code-gen / data-extraction / reasoning) or type your own.
  3. Toggle **Tool mode** on if you want `TOOL_CALL` spans to appear.
  4. Click **Run agent** and watch the span summary and performance chip appear.
- Card **How it works**: one paragraph on submit → agent-run → span-collect → flush → monitor; one paragraph on the two governance mechanisms (deployer-runtime monitor, periodic performance monitor).
- Card **Components**: rows per component (ConversationEntity, SpanCollector, TraceExportWorkflow, TracedConversationAgent, SpanInstrumentor, PerformanceMonitor, OtlpExporter, ConversationView, ConversationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus SpanInstrumentor + PerformanceMonitor + OtlpExporter as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `decisions_surface: no-decisions` and `oversight.human_on_loop: true` declarations are prominent. The `span_data_contains_prompt_text: true` field appears in the Data section. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, E1). ID badges coloured: H1 orange (hotl / runtime-monitor), E1 blue (eval-periodic).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Run a prompt. <span class="accent">See the trace.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Prompt preset` (Code generation / Data extraction / Reasoning / Custom), `Tool mode` toggle (when on, includes a tool-call span in the run), `Custom prompt` textarea (visible when Custom selected), `Submitted by` text input, and a teal **Run agent** button.
    - Live list below: one card per conversation, newest-first. Each card shows status pill, performance chip (quality score 1–5 badge, green/yellow/red), and age.
  - **Right column** — Selected-conversation detail.
    - Header: status pill + quality-score chip + `promptText` (first 60 chars + ellipsis).
    - Prompt preview: full prompt text in a monospace block.
    - Agent answer: the `answerText` in a prose block.
    - Span summary section: stat tiles for `totalSpans`, `llmTurnCount`, `toolCallCount`, `errorCount`, `wallClockMillis`. Then a span tree: each `SpanRecord` rendered as a collapsible row showing kind badge, `operationName`, duration (ms), and `status` badge. Child spans indented under their parent by `parentSpanId`.
    - Flush result section: `spansExported` / `spansDropped` counters; `exporterHealthy` badge (green "healthy" / yellow "degraded"); a clickable **View in Zipkin** link for `zipkinTraceUrl` (opens new tab).
    - Performance monitor section: a 1–5 star widget, p50/p95 latency figures, tool-success-rate bar (0–100%), and the one-line `rationale`. Quality score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, RUNNING=yellow, COMPLETED=blue, FLUSHED=purple, MONITORED=green, FAILED=red.
- SpanKind badge colours: LLM_CALL=teal, TOOL_CALL=orange, MEMORY_READ=muted-blue, MEMORY_WRITE=muted-purple, WORKFLOW_STEP=muted-green.
- SpanStatus badge colours: OK=green, ERROR=red, UNSET=muted.

The full raw span JSON is never displayed by default — the UI shows the span tree abstraction. A developer who needs the raw list can expand each span row to see its `attributes` map, or fetch `GET /api/conversations/{id}` for the complete `agentResponse.spans` array.
