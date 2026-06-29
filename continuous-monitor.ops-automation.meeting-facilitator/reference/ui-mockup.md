# UI mockup — meeting-facilitator

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Teams Facilitator</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed during development must be deleted from the HTML; `display:none` is insufficient. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Teams <span class="accent">Facilitator</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to drop a session, review the published summary, check the guardrail verdict badge).
- Card **How it works**: one paragraph on the poll → sanitize → summarize → guardrail → publish flow; one paragraph on the two governance mechanisms.
- Card **Components**: rows per component (poller, queue, sanitizer, notes agent, chat agent, workflow, entity, view, evalrunner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents, 1 AutonomousAgent, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `pii: true` and `human_on_loop: true` declarations are the most distinctive. Many deployer fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 cyan (sanitizer), G1 yellow (guardrail).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch sessions arrive. <span class="accent">Guardrail protects every summary.</span>`. Subtitle: `Simulated segments drop every 20 s. Published summaries passed both the sanitizer and the response guardrail.`
- Layout: two-column.
  - **Left column** — Live session list, sorted newest-first. Each card shows:
    - Header: status pill, guardrail verdict badge (PASS / FAIL), session age.
    - Sanitized transcript snippet (first 120 characters, truncated).
    - PII categories found (small muted chips).
  - **Right column** — the selected session detail.
    - Sanitized transcript (read-only, monospace, scrollable).
    - Meeting notes section: title, key points (bulleted), action items (bulleted).
    - Chat recap section: summary text, message count.
    - Guardrail verdict section: passed badge + reason text.
    - Eval score chip (when populated) + rationale.
- Status pill colours: ACTIVE=muted, SANITIZED=muted, SUMMARIZED=blue, PUBLISHED=green, SUPPRESSED=red.
- Guardrail verdict badge: PASS=green outline, FAIL=red filled.

Sessions in SUPPRESSED state show the guardrail `reason` prominently in the detail pane so the reviewer can understand why the summary was not published.
