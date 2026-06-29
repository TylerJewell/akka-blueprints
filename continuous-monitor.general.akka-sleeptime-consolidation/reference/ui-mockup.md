# UI mockup — sleeptime-consolidation

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Sleeptime Memory-Consolidation Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Sleeptime Memory-<span class="accent">Consolidation Agent</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, add some memory content, wait for consolidation to trigger, observe the drift score).
- Card **How it works**: one paragraph on the step-counter → trigger → consolidation cycle; one paragraph on the three governance mechanisms (guardrail, HOTL stream, drift eval).
- Card **Components**: rows per component (trigger, counter entity, block entity, workflow, primary agent, consolidator agent, drift eval agent, view, drift runner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents (typed), 1 AutonomousAgent, 1 Workflow, 2 ESEs, 1 View, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs with answers from `risk-survey.yaml`. The `human_on_loop: true` and `audit_frequency: continuous` declarations are the most distinctive for this domain. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, E1). ID badges coloured: G1 yellow (guardrail), H1 pink (hotl), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch memory evolve. <span class="accent">Catch drift before it matters.</span>`. Subtitle: `Consolidation runs every N steps. Drift is scored automatically.`
- Layout: two-column.
  - **Left column** — Live block list, sorted newest-first. Each card shows:
    - Header: status pill, version badge, block age.
    - Topic hint chips (small, muted).
    - Drift score badge (colour-coded: green ≤ 50, amber 51–69, red ≥ 70).
  - **Right column** — selected block detail.
    - Consolidated text (read-only, monospace).
    - Change rationale (muted text).
    - HOTL activity stream: scrolling chronological log of `ConsolidationStarted`, `BlockConsolidated`, `DriftScored`, and `GuardrailTriggered` events for this block. Guardrail trips are red rows; high-drift scores are amber rows; successful consolidations are green rows.
    - Drift score with rationale (when available).
    - Prior summary / current summary side-by-side (when drift is scored).
- Status pill colours: ACTIVE=blue, CONSOLIDATING=amber, CONSOLIDATED=green, STALE=muted/red.
- A "Add content" button opens a small textarea for manually adding raw content to the selected block (calls `POST /api/memory/blocks/{blockId}/content`).
- A "Step counter" widget in the header shows current `stepsSinceLastConsolidation` / threshold for the active session.
