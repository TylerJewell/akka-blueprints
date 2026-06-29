# UI mockup — live-eval-harness

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Live Evals</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Live <span class="accent">Evals</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to drop a decision, observe the rubric score, check the drift panel for aggregate status).
- Card **How it works**: one paragraph on the poll → sanitize → eval → alarm-check flow; one paragraph on the two governance mechanisms (on-decision eval and periodic drift watch).
- Card **Components**: rows per component (poller, queue, sanitizer, rubric-eval agent, drift-watch agent, workflow, decision entity, drift snapshot entity, drift sampler, eval view, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 1 AutonomousAgent, 1 Workflow, 3 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model).
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs with answers from `risk-survey.yaml`. The `user_identifiable_fields_stripped_before_eval: true` declaration in Data is the most distinctive chip. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (E1, E2). ID badges coloured: E1 green (eval-event), E2 green-teal (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch every decision <span class="accent">get scored.</span>`. Subtitle: `Simulated decisions arrive every 10 s. Drift status updates every 15 minutes.`
- Layout: two-column.
  - **Left column** — Live decision feed, sorted newest-first. Each card shows:
    - Header: status pill, overall score badge (1–5), agent ID label.
    - Sanitized input summary (truncated to one line).
    - Alarm indicator if `status=ALARMED` (red pill).
  - **Right column** — selected decision detail plus a drift status panel at the top.
    - **Drift panel** (always visible, top of right column): `DriftStatus` badge (OK=green, WATCH=yellow, ALARM=red), narrative, flagged dimensions chips, last assessed timestamp.
    - Sanitized input (read-only, monospace, collapsible).
    - Sanitized output (read-only, monospace, collapsible).
    - Rubric dimensions table: dimension name | score badge | justification. One row per dimension.
    - Overall score large badge + summary sentence.
    - Eval timestamp and model version.
- Status pill colours: RECEIVED=muted, SANITIZED=muted, EVALUATED=blue, OK=green, ALARMED=red.
- Score badge colours: 5=green, 4=green-teal, 3=yellow, 2=orange, 1=red.
- Drift status badge colours: OK=green, WATCH=yellow, ALARM=red.

There are no approve/reject actions — this system is monitoring-only. Governance officers respond out-of-band.
