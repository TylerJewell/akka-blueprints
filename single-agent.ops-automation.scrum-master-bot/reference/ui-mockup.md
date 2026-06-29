# UI mockup — scrum-master-bot

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: ScrumMasterBot</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Scrum<span class="accent">MasterBot</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded sprint fixture (Feature Team / Platform Team / Release Team) or enter a custom roster.
  3. Click **Run standup**.
  4. Watch the card transition through COLLECTING → RUNNING → SUMMARY_READY → POSTED.
- Card **How it works**: one paragraph on activate → collect-context → run-standup → post-updates; one paragraph on the guardrail mechanism (scope-gated ticket writes).
- Card **Components**: rows per component (StandupEntity, SprintConsumer, StandupWorkflow, ScrumMasterAgent, TicketWriteGuardrail, TicketPostingService, StandupView, StandupEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Service as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = automated-with-scope-gate` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Run a standup. <span class="accent">Post the summary.</span>`. Subtitle: `One agent, one scope-gated tool.`
- Layout: two-column.
  - **Left column** — Session panel + live list.
    - Session panel: dropdown `Sprint fixture` (Feature Team / Platform Team / Release Team / custom), `Sprint ID` text input, `Team name` text input, `Members` editable list (memberId, displayName, role per row), `Authorized ticket IDs` tag input, and a yellow `Run standup` button.
    - Live list below: one card per session, newest-first. Each card shows status pill, outcome badge (when summary landed), team name, age.
  - **Right column** — Selected-session detail.
    - Header: status pill + outcome badge + sprint name + team name.
    - Sprint context: sprint number, sprint dates, member count, ticket count.
    - Per-member rows: member display name and role, yesterday text, today text, blocker text (highlighted orange if present).
    - Ticket updates: a table with ticket id, posted-to badge (green checkmark if posted, red x if skipped), comment preview.
    - Summary text: the agent's 2–4-sentence paragraph.
    - Next actions list: bulleted with actionable-verb style.
    - Outcome section at bottom: outcome badge (ON_TRACK=green, AT_RISK=yellow, BLOCKED=red) with the summary text.
- Status pill colours: COLLECTING=muted, RUNNING=yellow, SUMMARY_READY=blue, POSTED=green, FAILED=red.
- Outcome badge colours: ON_TRACK=green, AT_RISK=yellow, BLOCKED=red.
- Blocker field: orange background if non-null.
