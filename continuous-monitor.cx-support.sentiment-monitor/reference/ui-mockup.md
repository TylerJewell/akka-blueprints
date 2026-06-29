# UI mockup — sentiment-monitor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Sentiment Monitor</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Sentiment <span class="accent">Monitor</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for simulated comments to arrive, watch for a thread to reach CRITICAL, observe the Slack alert dispatch).
- Card **How it works**: one paragraph on the poll → score → trend-check → alert flow; one paragraph on the two governance mechanisms (guardrail + drift watch).
- Card **Components**: rows per component (poller, queue, consumer, scorer, analyst, scoring workflow, alert workflow, thread entity, eval entity, thread view, eval runner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 1 AutonomousAgent, 2 Workflows, 3 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model) from `PLAN.md`.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `decisions_surface: alert-only` declaration is the most distinctive — no individual customer data is acted upon directly. `oversight.human_on_loop = true` surfaces as filled chip. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 yellow (guardrail), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the threads. <span class="accent">Catch the drift.</span>`. Subtitle: `Simulated Linear comments arrive every 20 s. Threads turn CRITICAL when consecutive negative scores cross the threshold.`
- Layout: two-column.
  - **Left column** — Live thread list, sorted CRITICAL → DECLINING → STABLE. Each card shows:
    - Trend badge: CRITICAL (red), DECLINING (orange), STABLE (green).
    - Issue title and ID.
    - Running average chip (e.g., `avg −3.2`).
    - Consecutive negative count (e.g., `6 consecutive negative`).
    - Thread status pill: ACTIVE / SILENCED / RESOLVED.
  - **Right column** — the selected thread detail.
    - Last 10 scored comments, newest-first. Each comment row:
      - Score chip (−5 to +5, coloured red→neutral→green).
      - Confidence label (`high` / `medium` / `low`).
      - Score reason (muted text).
      - Comment body (read-only, truncated at 200 chars).
    - Running average and trend label.
    - Silence button (visible when status is ACTIVE and trendLabel is CRITICAL). Resolve button.
    - Silence opens a small confirmation; resolve is a single click.
    - If `lastAlert.dispatched == true`, show a chip: "Alert sent to #channel".
    - Eval drift badge (when `SentimentEvalEntity` has a recent result): NO_DRIFT (muted), MINOR_DRIFT (yellow), SIGNIFICANT_DRIFT (red).
- Score chip colour scale: −5 to −3 = red, −2 to −1 = orange, 0 = neutral/grey, +1 to +2 = light green, +3 to +5 = green.
