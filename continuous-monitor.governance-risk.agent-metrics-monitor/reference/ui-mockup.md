# UI mockup — agent-metrics-monitor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Agent Metrics Monitor</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Agent Metrics <span class="accent">Monitor</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for first batch, inspect a health card, watch the eval score appear).
- Card **How it works**: one paragraph on the poll → ingest → detect → narrate loop; one paragraph on the eval-periodic governance mechanism.
- Card **Components**: rows per component (poller, queue, ingestor, detector, narrator, evaluator, workflow, entity, view, evalrunner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents, 1 Workflow, 2 ESEs, 1 Consumer, 2 TimedActions, 1 View, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model) using Akka theme variables and Lesson 24 CSS overrides.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `pii: false` declaration in Data is notable — no PII flows through this system. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (E1). ID badge coloured green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Monitor agent health. <span class="accent">Catch anomalies as they land.</span>`. Subtitle: `New metric batches arrive every 60 s. Eval scores update every 30 minutes.`
- Layout: two-column.
  - **Left column** — Agent health card list, sorted by severity (CRITICAL first, then DEGRADED, then HEALTHY). Each card shows:
    - Header: status badge (colour-coded), agent name, `lastUpdatedAt` age.
    - Key metric chips: error rate, p99 latency (muted until above threshold; coloured when breaching).
    - Anomaly signals as small muted chips (hidden when HEALTHY).
    - Summary excerpt (first sentence of `latestSummary.narrativeText`).
    - Eval score chip (hidden until scored).
  - **Right column** — Selected agent detail panel.
    - Current status badge + agent ID.
    - Metric history table: one row per `AgentMetricSample` in `recentSamples`, columns = windowEnd, invocations, errorRate, p50ms, p99ms. Rows coloured by threshold breach.
    - Latest anomaly section: signals list, verdict sentence.
    - Latest summary section: full `narrativeText`, producedAt timestamp, eval score chip + rationale when present.
- Status badge colours: HEALTHY=green, DEGRADED=yellow, CRITICAL=red.
- Metric chip colours: within threshold=muted, warning breach=yellow, critical breach=red.
