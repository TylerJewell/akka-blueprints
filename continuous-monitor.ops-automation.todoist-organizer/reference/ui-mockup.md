# UI mockup — todoist-organizer

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Todoist AI Inbox Organizer</title>`.

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
- Headline: `Todoist AI Inbox <span class="accent">Organizer</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to drop a task, review the classification and guardrail verdict, observe UPDATED or SKIPPED status).
- Card **How it works**: one paragraph on the poll → classify → guardrail → update flow; one paragraph on the two governance mechanisms.
- Card **Components**: rows per component (poller, queue, consumer, classifier, workflow, entity, view, evalrunner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `human_in_loop: false` / `automated_guardrail: true` pairing is the most distinctive — the system relies on the guardrail rather than a human approval gate. Deployer-specific fields are faded with `TO_BE_COMPLETED_BY_DEPLOYER`.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 yellow (guardrail), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch tasks get organized. <span class="accent">Guardrail catches low-confidence results.</span>`. Subtitle: `Simulated tasks arrive every 30 s. The classifier routes each one; the guardrail decides whether to write.`
- Layout: two-column.
  - **Left column** — Live task list, sorted newest-first. Each card shows:
    - Header: status pill, confidence chip, task age.
    - Task content (title).
    - Target project name and label chips (when classified).
    - Guardrail verdict chip (allowed / blocked).
  - **Right column** — the selected task detail.
    - Original task content (read-only).
    - Classification breakdown: target project, labels, priority, confidence, reason.
    - Guardrail verdict: allowed boolean and reason.
    - Update record (when UPDATED): project written, labels applied, priority, timestamp.
    - Eval score chip (when populated) and rationale.
- Status pill colours: FETCHED=muted, CLASSIFIED=blue, GUARDRAIL_BLOCKED=red, UPDATED=green, SKIPPED=yellow, FAILED=red.
- Confidence chip colours: high=green, medium=yellow, low=red.
- Guardrail verdict chip colours: allowed=green, blocked=red.
