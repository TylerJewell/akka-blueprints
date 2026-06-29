# UI mockup — churn-monitor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Customer Churn Prediction Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Customer Churn <span class="accent">Prediction Agent</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the poller to drip an account, review the churn score and retention plan, click Mark Actioned or Dismiss).
- Card **How it works**: one paragraph on the poll → sanitize → score → advise → manager-review flow; one paragraph on the two governance mechanisms (PII sanitizer + drift/fairness eval).
- Card **Components**: rows per component (poller, queue, sanitizer, scorer, advisor, workflow, entity, view, evalrunner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 1 AutonomousAgent, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards (component graph, interaction sequence, state machine, entity model).
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled; `human_in_loop: true` (HIGH risk only) is filled; many deployer-specific fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, E1). ID badges coloured: S1 cyan (sanitizer), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Live churn risk board. <span class="accent">Act on every HIGH-risk account.</span>`. Subtitle: `Account snapshots arrive every 60 s. LOW and MEDIUM risk accounts close automatically.`
- Layout: two-column.
  - **Left column** — Live account risk board, sorted HIGH first then by creation time. Each card shows:
    - Header: status pill, risk level badge (HIGH=red, MEDIUM=yellow, LOW=green), snapshot age.
    - Redacted account name (never the raw name).
    - Industry + tier chips.
    - Churn probability bar (0–100%).
  - **Right column** — selected account detail.
    - Top signals list (from `ChurnScore.topSignals`).
    - Retention plan actions (numbered, with actionType badge and description) — only shown for HIGH-risk accounts.
    - **Mark Actioned** button (yellow). **Dismiss** button (border). Dismiss opens a small reason textarea.
    - After eval: drift and fairness chips showing score + verdict.
    - After action/dismiss: the decision metadata (decidedBy, decidedAt, reason if present).
- Status pill colours: RECEIVED=muted, SANITIZED=muted, SCORED=blue, ADVISED=blue, AWAITING_ACTION=yellow, ACTIONED=green, DISMISSED=muted, LOW_RISK_CLOSED=muted, MEDIUM_RISK_CLOSED=muted.
- Risk level badge colours: HIGH=red fill, MEDIUM=yellow fill, LOW=green fill.

The left column list refreshes via SSE (`GET /api/churn/sse`). Selecting an account fetches the full detail via `GET /api/churn/{accountId}`.
