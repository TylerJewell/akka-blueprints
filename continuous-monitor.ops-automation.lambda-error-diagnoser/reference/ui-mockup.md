# UI mockup — lambda-error-diagnoser

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Lambda Error Diagnoser</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Lambda Error <span class="accent">Diagnoser</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to drop an error event, inspect the diagnosis and eval score, optionally dismiss the incident).
- Card **How it works**: one paragraph on the poll → normalise → diagnose → eval → resolve flow; one paragraph on the on-incident eval mechanism and why it fires synchronously inside the workflow.
- Card **Components**: rows per component (poller, queue, normalizer, diagnosis agent, eval judge, workflow, entity, view, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `human_in_loop: false` declaration in Oversight is the most distinctive — the chip is filled amber with a warning. `decisions_surface: advisory-only` is pre-filled. Many deployer-specific fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (E1). ID badge coloured green (eval-event).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the incident board. <span class="accent">Diagnoses arrive with scores.</span>`. Subtitle: `Error logs drip every 20 s. Every diagnosis is scored before it surfaces.`
- Layout: two-column.
  - **Left column** — Live incident list, sorted by severity (CRITICAL first) then newest-first within each severity band. Each card shows:
    - Header: severity badge, status pill, incident age.
    - Function name in monospace.
    - Error category chip and confidence chip.
    - Eval score chip (1–5, coloured by score: 5=green, 3–4=yellow, 1–2=red).
  - **Right column** — selected incident detail.
    - Normalised error fields: errorCategory, errorMessage, coldStart flag.
    - Stack trace excerpt (read-only, monospace, scrollable).
    - Diagnosis block: rootCause (full text), fixSuggestion, reasoning, confidence.
    - Eval block: score badge, rationale.
    - Dismiss button (border, muted). Dismiss opens a reason textarea. Re-open button if status = DISMISSED.
- Status pill colours: DETECTED=muted, NORMALISED=muted, DIAGNOSED=blue, EVALUATED=blue, RESOLVED=green, DISMISSED=muted, REOPENED=yellow.
- Severity badge colours: CRITICAL=red, HIGH=orange, MEDIUM=yellow, LOW=muted.

Dismissing does not delete the incident. The card remains on the board with the DISMISSED badge and the reason visible in the detail pane.
