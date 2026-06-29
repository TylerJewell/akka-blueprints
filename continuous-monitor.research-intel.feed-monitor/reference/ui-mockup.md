# UI mockup — feed-monitor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Feed Monitor</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index.

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed during iteration must be deleted from the HTML; `display:none` is not sufficient.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Feed <span class="accent">Monitor</span>`. No subtitle.
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for a feed item to appear, review classification and summary, act on PENDING_REVIEW items).
- Card **How it works**: one paragraph on the poll → summarize → classify → notify/suppress/review flow; one paragraph on the two governance mechanisms (channel guardrail + eval).
- Card **Components**: rows per component (poller, entity, summary agent, classifier agent, workflow, notifier, view, evalrunner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents, 1 AutonomousAgent, 1 Workflow, 1 ESE, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `human_on_loop: true` declaration in Oversight is the most distinctive. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 yellow (guardrail), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the feed. <span class="accent">Act on reviews.</span>`. Subtitle: `Feed items arrive every 60 s. REVIEW items wait for your decision.`
- Layout: two-column.
  - **Left column** — Live feed list, sorted newest-first. Each card shows:
    - Header: status pill, classification chip, item age.
    - Feed title (truncated at 80 characters).
    - Topics as small muted chips.
    - Eval score chip when populated.
  - **Right column** — the selected item detail.
    - Feed source URL (read-only).
    - AI summary (read-only, prose).
    - Extracted topics (chips).
    - Key quote (monospace, bordered).
    - Classification reason (muted text).
    - When status is PENDING_REVIEW: Override button (yellow) + Suppress button (border). Override opens a channel input pre-filled with `"#research-intel"`.
    - When status is POSTED: Slack post record (channel, postedAt, guardrailPassed badge).
    - When eval populated: eval score chip + rationale.
- Status pill colours: RECEIVED=muted, SUMMARIZED=muted, CLASSIFIED=blue, NOTIFY_REQUESTED=yellow, POSTED=green, SUPPRESSED=muted, PENDING_REVIEW=yellow, FAILED=red.

Override and Suppress actions call `POST /api/feed/{id}/override` and `POST /api/feed/{id}/suppress` respectively; the UI listens on the SSE stream for the resulting status transition.
