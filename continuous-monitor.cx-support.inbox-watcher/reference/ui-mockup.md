# UI mockup — inbox-watcher

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Inbox Watcher</title>`.

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

And the DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents (clicking a tab and seeing a blank because a hidden zombie panel occupied the index).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Inbox <span class="accent">Watcher</span>`. **No subtitle.**
- Card **Try it**: env-var instruction + `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to drop a message, review the draft, click Approve / Reject).
- Card **How it works**: one paragraph on the poll → sanitize → classify → draft → approve flow; one paragraph on the four governance mechanisms.
- Card **Components**: rows per component (poller, queue, sanitizer, filter, drafter, workflow, entity, view, evalrunner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 1 AutonomousAgent, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `pii: true` declaration in Data is the most distinctive — the chip is filled. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with four rows (S1, G1, H1, E1). ID badges coloured: S1 cyan (sanitizer), G1 yellow (guardrail), H1 pink (hitl), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the inbox. <span class="accent">Approve every reply.</span>`. Subtitle: `Simulated messages drop every 15 s. Drafts are held until you act.`
- Layout: two-column.
  - **Left column** — Live message list, sorted newest-first. Each card shows:
    - Header: status pill, classification chip, message age.
    - Sanitized subject (the user never sees the raw subject).
    - PII categories found (small muted chips).
  - **Right column** — the selected message detail.
    - Sanitized body (read-only, monospace).
    - Classification reason.
    - Drafted reply (textarea — editable).
    - Approve button (yellow). Reject button (border). Reject opens a small "Reason" textarea.
    - After action, the eval score chip (when populated).
- Status pill colours: RECEIVED=muted, SANITIZED=muted, CLASSIFIED=blue, DRAFTED=blue, AWAITING_APPROVAL=yellow, SENT=green, DISMISSED=muted, ESCALATED=red.

Editing the draft body before Approve is allowed — the approved body is what is logged as "what would have been sent."
