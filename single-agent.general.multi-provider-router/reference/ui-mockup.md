# UI mockup — multi-provider-router

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: MultiProviderRouter</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The exemplar's `static-resources/index.html` has the canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Multi<span class="accent">Provider</span>Router`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded prompt or type your own. Leave the provider hint on `auto` for the first call.
  3. Click **Send**.
  4. Watch the card transition through PENDING → DISPATCHED → COMPLETED, then check which provider handled the call.
- Card **How it works**: one paragraph on submit → dispatch → record; one paragraph on the periodic performance monitor and what the Eval Matrix tab shows.
- Card **Components**: rows per component (CallEntity, RoutingWorkflow, RouterAgent, PerformanceMonitor, EvaluationAggregator, CallView, MonitorView, RouterEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 2 Views, 1 Consumer, 2 HttpEndpoints, plus 1 Aggregator as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `decisions.authority_level = informational` and `oversight.human_in_loop = false` are the distinctive answers — this is a fully automated dispatch system, not an advisory one. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (E1). ID badge coloured blue (eval-periodic).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.
- Below the control table: a **Live Monitor** panel that renders the latest `MonitorRow` data from `GET /api/monitor`. Shows per-provider: a provider badge, P50 latency bar, P95 latency bar, error rate, and quality score chip (1–5 coloured green/yellow/red). Refreshes on SSE updates.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a prompt. <span class="accent">See which provider answered.</span>`. Subtitle: `One agent, two backends, one performance monitor.`
- Layout: two-column.
  - **Left column** — Submission panel + live call list.
    - Submission panel: `Prompt` textarea (with a "Load seeded example" link that fills the textarea), `Provider hint` radio group (`auto` / `openai` / `anthropic`), `Submitted by` text input, and a yellow `Send` button.
    - Live call list below: one card per call, newest-first. Each card shows status pill, provider badge (when dispatched), latency chip (when completed), prompt text truncated to 60 chars, age.
  - **Right column** — Selected-call detail.
    - Header: status pill + provider badge + latency chip + prompt text (first 120 chars).
    - Prompt text block: full prompt in a monospace box.
    - Provider section: the selected provider name and the hint used.
    - Response text: the agent's answer in a prose block.
    - Performance context: the latest `ProviderStats` for this call's provider (P50, P95, quality score chip), fetched from `GET /api/monitor`.
- Status pill colours: PENDING=muted, DISPATCHED=yellow, COMPLETED=green, FAILED=red.
- Provider badge colours: OPENAI=blue, ANTHROPIC=purple, MOCK=grey.
- Quality score chip: 4–5=green, 3=yellow, 1–2=red.
- Latency chip: <500ms=green, 500–1500ms=yellow, >1500ms=red.

The prompt text is shown in full in the right pane — no redaction occurs in this blueprint (the risk-survey declares `pii: TO_BE_COMPLETED_BY_DEPLOYER`; a deployer handling PII would wire a sanitizer as a separate component before the agent call).
