# UI mockup — workflow-observability

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Workflow Observability</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in a prior iteration must be deleted from the HTML, not hidden with `display:none`. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Workflow <span class="accent">Observability</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to emit an item, inspect the span waterfall, check the eval score after 30 min or reduced interval).
- Card **How it works**: one paragraph on the poll -> route -> process -> span-export flow; one paragraph on the two governance mechanisms (deployer-runtime-monitoring and periodic eval).
- Card **Components**: rows per component (poller, queue, collector, router, summariser, workflow, entity, exporter, view, evalsampler, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 1 AutonomousAgent, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model).
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `autonomous` authority level and `human_on_loop: true` declarations are the most distinctive. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (M1, E1). ID badges coloured: M1 pink (hotl / deployer-runtime-monitoring), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the traces. <span class="accent">Monitor every agent run.</span>`. Subtitle: `Simulated workload items arrive every 20 s. Spans accumulate in real time.`
- Layout: two-column.
  - **Left column** — Live item list, sorted newest-first. Each card shows:
    - Header: status pill, routing chip (SUMMARISE / ESCALATE / SKIP), item age.
    - `workloadType` badge and `sourceTag`.
    - Payload preview (first 80 characters).
  - **Right column** — selected item detail.
    - Span waterfall: one row per span with operation name, duration bar, token counts, and status dot.
    - Routing rationale (read-only, italic).
    - Summary output (read-only, monospace): `summary` and `keyFindings`.
    - Eval score chip (1-5, coloured) and one-sentence rationale — visible only when `eval` is populated.
- Status pill colours: QUEUED=muted, ROUTED=blue, PROCESSING=blue, EXPORTED=green, EVALUATED=cyan, ESCALATED=red, SKIPPED=muted, FAILED=red.
- Routing chip colours: SUMMARISE=green, ESCALATE=yellow, SKIP=muted.
- The span waterfall duration bar is a horizontal progress bar scaled to the longest span in the item. Token counts appear as `P:210 C:92` in a muted monospace font.
