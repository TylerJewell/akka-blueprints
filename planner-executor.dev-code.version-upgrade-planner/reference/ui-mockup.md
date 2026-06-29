# UI mockup — version-upgrade-planner

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from Section 13.

Browser title: `<title>Akka Sample: Airflow Version Upgrade Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index (Lesson 26):

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements. Any panel removed during iteration must be deleted from the HTML — `display:none` is not sufficient; it leaves a zombie panel that will be activated when a later tab matches its index.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the mermaid CSS overrides AND `themeVariables` — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these, state names render invisible (black-on-black) and edge labels clip on the Architecture tab.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Airflow Version <span class="accent">Upgrade Agent</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit an upgrade job in the App UI tab, approve the migration phases when prompted, watch the job progress through compatibility check, migration, and test run.
- Card **How it works**: one paragraph naming the components, the loop steps, the three executors, the approval gate, and the CI test gate.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the approval endpoints and `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (4 agents, 1 workflow, 4 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text and the answer drawn from `risk-survey.yaml`. Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `hitl` amber (`HI1`), `ci-gate` blue (`CI1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an upgrade job. <span class="accent">Approve phases. Watch it run.</span>` Subtitle: `The simulator drips a job every 120 s so the page is never empty.`
- Form card: two text fields labelled "Source version" and "Target version", optional "Requested by" field, `Submit` button (yellow).
- **Approval pane** (shown when any job is `AWAITING_APPROVAL`): a prominently-styled card showing the pending approval request — phase name, rationale, requested-at timestamp. Two buttons: `Approve` (yellow) and `Reject` (muted, opens a required comment field before confirming). The pane disappears when no approval is pending.
- **Operator controls pane**: a card showing the current halt state.
  - If `halted=false`: yellow `Halt new phases` button, a free-text reason field.
  - If `halted=true`: muted `HALTED` pill with the reason and timestamp, plus a `Resume` button.
  - Reflects every `control-update` SSE event live.
- Live list: cards per job; left border coloured by status (PLANNING = muted, EXECUTING = blue, AWAITING_APPROVAL = amber, COMPLETED = green, FAILED = red, HALTED = orange, STUCK = pale red).
  - Header row: source→target version string, status pill, phase count, elapsed time.
  - Click to expand:
    - **Plan ledger**: phase list as a numbered stepper — each step shows the phase name, kind badge (COMPAT_CHECK green, MIGRATION amber, TEST_RUN blue), and `requires approval` icon when applicable. The current phase is highlighted.
    - **Progress ledger**: vertical timeline of `ProgressEntry` rows. Each row shows attempt number, executor pill (`COMPAT_CHECKER` green, `TEST_RUNNER` blue, `MIGRATION_APPLIER` amber), phase name, verdict pill (`OK` green, `BLOCKED_BY_APPROVAL` amber, `CI_GATE_FAILED` red, `FAILED` dark red), and a collapsed `<pre>` of `summary` (click to expand). Rows with a non-null `testReport` show a mini test-result bar (passed/failed count) inline.
    - **Pending approval** (only when status is `AWAITING_APPROVAL`): duplicates the approval pane content inline within the expanded job card, including the Approve/Reject buttons.
    - **Upgrade report** (only when status is `COMPLETED`): summary paragraph + applied phases list + tests-passed badge.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
