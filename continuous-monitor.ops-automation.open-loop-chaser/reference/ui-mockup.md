# UI mockup — open-loop-chaser

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Intelligent Reminders</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed during iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Intelligent <span class="accent">Reminders</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to drop source events, confirm an item's owner, observe the nudge dispatch).
- Card **How it works**: one paragraph on the poll → sanitize → extract → track → nudge flow; one paragraph on the two governance mechanisms.
- Card **Components**: rows per component (poller, queue, sanitizer, extractor, composer, extraction workflow, nudge workflow, entity, view, scanner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 1 AutonomousAgent, 2 Workflows, 2 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `pii: true` declaration in Data is the most distinctive — the chip is filled. The `notify-only` authority level is prominent. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 cyan (sanitizer), G1 yellow (guardrail).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Track open loops. <span class="accent">Confirm every nudge.</span>`. Subtitle: `Simulated source events arrive every 20 s. Nudges wait until you confirm the owner.`
- Layout: two-column.
  - **Left column** — Live action item list, sorted newest-first. Each card shows:
    - Header: status pill, source type chip (MEETING_NOTES / MESSAGE / DOCUMENT), item age.
    - Sanitized description (the user never sees raw source text).
    - Suggested owner (grayed if not yet confirmed), confirmed owner if set.
    - Nudge count badge.
  - **Right column** — the selected item detail.
    - Sanitized source text excerpt (read-only, monospace).
    - Extracted action item description.
    - Confirm Owner field: text input + Confirm button (yellow).
    - Close button (green). Snooze button (border) — opens a duration picker.
    - Nudge history section showing composed nudges in reverse order.
- Status pill colours: DETECTED=muted, SANITIZED=muted, PENDING=blue, STALE=yellow, NUDGED=green, SNOOZED=muted, CLOSED=muted, FAILED=red.

Operator workflow: items arrive in PENDING. Confirm the owner so the nudge path is unblocked. Once stale, the nudge fires automatically. If the item is resolved, mark it CLOSED. If it is temporarily blocked, snooze it.
