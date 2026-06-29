# UI mockup — slack-lead-qualifier

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Slack Lead Qualifier</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is insufficient. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Slack Lead <span class="accent">Qualifier</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for simulator to drop a member event, watch the lead enrichment and score, see the Slack post or suppression result).
- Card **How it works**: one paragraph on the poll → enrich → sanitize → score → post flow; one paragraph on the two governance mechanisms (PII sanitizer + guardrail).
- Card **Components**: rows per component (poller, queue, enrichment agent, scoring agent, sanitizer, workflow, entity, view, endpoint) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model).
- Mermaid diagrams must include the CSS overrides and theme variables from Lesson 24 (state-diagram label colour, edge-label foreignObject `overflow:visible`, `transitionLabelColor #cccccc`).
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs with answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled. `human_in_loop: false` / `human_on_loop: true` shows the automated-post-with-oversight distinction. `TO_BE_COMPLETED_BY_DEPLOYER` fields faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 cyan (sanitizer), G1 yellow (guardrail).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch leads arrive. <span class="accent">See every post or block.</span>`. Subtitle: `Simulated member events drop every 20 s. Each lead is enriched, scored, and either posted or suppressed automatically.`
- Layout: two-column.
  - **Left column** — Live lead queue, sorted newest-first. Each card shows:
    - Header: status pill, tier chip (HOT/WARM/COLD/DISQUALIFIED), score badge, event age.
    - Company and job title (from sanitized enrichment).
    - PII-categories-stripped chips (small, muted).
  - **Right column** — selected lead detail:
    - Enriched company, title, industry, headcount (read-only).
    - Sanitized enrichment diff — fields stripped (muted list).
    - Score breakdown: matched criteria (green chips), missed criteria (muted chips).
    - Scoring rationale (paragraph).
    - Slack post draft: channel, headline, body (read-only).
    - Guardrail result: APPROVED (green) or BLOCKED (red) with violation reason.
    - Suppression reason when applicable.
- Status pill colours: RECEIVED=muted, ENRICHED=blue, SANITIZED=blue, SCORED=blue, POSTING=yellow, POSTED=green, SUPPRESSED=muted, FAILED=red.
- Tier chip colours: HOT=red, WARM=yellow, COLD=muted, DISQUALIFIED=muted-strikethrough.
