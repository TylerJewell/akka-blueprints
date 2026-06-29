# UI mockup — campaign-optimizer-loop

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Campaign Optimizer</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index:

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

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram on the Architecture tab renders state names invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Campaign <span class="accent">Optimizer</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a campaign brief in the App UI tab, watch the planner iterate through specialist steps, approve the campaign as marketer, see it go live.
- Card **How it works**: one paragraph naming the components, the planning loop, the approval gate, and the four specialists.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/campaigns/{id}/approve` and `/api/campaigns/{id}/reject` routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (5 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text, drives sublabel, and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `hitl` blue (`H1`), `eval-periodic` green (`E1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a brief. <span class="accent">Watch it optimize.</span>` Subtitle: `The simulator drips a campaign every 120 s so the page is never empty.`
- Form card: fields for "Campaign goal", "Target audience", "Channels" (comma-separated), and `Submit` button (yellow).
- **Approval queue pane**: a card listing all campaigns in `AWAITING_APPROVAL` status. Each entry shows goal (first 80 chars), a blue `AWAITING APPROVAL` pill, and two buttons — `Approve` (yellow) and `Reject` (muted red). Clicking either opens an inline form for `decidedBy` and an optional `note`, then calls the appropriate endpoint.
- Live list: cards per campaign; left border coloured by status (PLANNING = muted, EXECUTING = blue, AWAITING_APPROVAL = amber, LIVE = teal, COMPLETED = green, FAILED = red, REJECTED = orange).
  - Header row: goal (first 80 chars), status pill, step count, elapsed time.
  - Click to expand:
    - **Campaign ledger**: goals (yellow list), target audience (muted list), channel plan (numbered list), brand constraints (red list), currentDispatch (specialist pill + step line).
    - **Run ledger**: vertical timeline of `RunEntry` rows. Each row shows attempt number, specialist pill (`COPY` purple, `AUDIENCE` blue, `PUBLISH` teal, `PERFORMANCE` green), step, verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red, `KPI_MISS` orange), metrics snapshot chips (openRate, clickRate, conversionRate shown when non-null), and a collapsed `<pre>` of `output` (click to expand).
    - **Report** (only when status is COMPLETED): summary paragraph + highlights bullets + final metrics tile.
    - **Rejection reason** (when status is REJECTED): one-paragraph block in orange.
    - **Failure reason** (when status is FAILED): one-paragraph block in red.
