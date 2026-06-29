# UI mockup — shared-memory-multi-agent

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Shared-Memory Multi-Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Shared-Memory <span class="accent">Multi-Agent</span>`. **No subtitle.**
- Card **Try it**: a `/akka:build` (Claude Code) block, then three numbered steps (seed a memory block, watch writers contribute fragments, trigger consolidation and see the merged snapshot). No env-var export block.
- Card **How it works**: one paragraph naming the components and the shared-write + consolidation coordination.
- Card **Components**: table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, event-sourced = yellow, key-value = gold, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, block state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind.
- Four mermaid cards (component graph, sequence, block state machine, ER) with the Akka theme variables AND the Lesson 24 state-label CSS overrides (white state names, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`). Without these the state names render black-on-black and the arrow labels clip.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sections in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each question block is rendered with the question text and the chips / textareas / list widgets in their selected state per `risk-survey.yaml`.
- Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer"); unanswered blocks get `opacity: 0.45`.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges carry a colored mechanism pill: guardrail = red, sanitizer = teal.
- Rows expand vertically on click to show rationale + implementation; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Seed a memory block. <span class="accent">Watch agents build it together.</span>` Subtitle: `The simulator also seeds a block every 90 s so the board is never empty.`
- Form card: a "Block name" field, an "Initial content" textarea, a `Seed` button (yellow).
- The board: one column per `BlockStatus` (Seeded · Active · Consolidating · Consolidated · Archived). Each block is a card showing its name, pending fragment count, last writer id, last consolidation summary (when present), and a `Consolidate` button (triggers on-demand pass). A `CONSOLIDATING` card shows a spinner.
- Fragment panel: clicking a block card opens a side panel listing all fragments for that block — each with the author id, content (first 80 chars), status chip (PENDING / SANITIZED / MERGED / REJECTED / REQUEUED), and a sanitized-note badge if the sanitizer fired.
- Writer roster strip: a compact list of the registered writer ids and their last-active timestamps, pulled from `GET /api/agents`.
- Cards move between columns live via SSE.
