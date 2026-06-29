# UI mockup — code-assistant-loop

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Agentic Code Assistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements. Any panel removed in an earlier iteration must be deleted; `display:none` is not sufficient (Lesson 26).

## Mermaid CSS overrides (Lesson 24)

The `<style>` block includes CSS overrides AND `themeVariables` — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these, state names render invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Agentic <span class="accent">Code Assistant</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a coding task in the App UI tab, watch the planner loop read, edit, and test, expand the row to see the plan ledger, edit log, and commit summary.
- Card **How it works**: one paragraph naming the components, the loop steps, and the three agent roles.
- Card **Components**: table with one row per component from `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (4 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text and the selected state from `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `ci-gate` blue (`E1`), `halt` red (`HT1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Watch the loop run.</span>` Subtitle: `The simulator drips a task every 90 s so the page is never empty.`
- Form card: text field labelled "Coding task", `Submit` button (yellow).
- **Operator controls pane** (top right of the App UI tab):
  - If `halted=false`: yellow `Halt new edits` button, a free-text reason field.
  - If `halted=true`: muted `HALTED` pill with the reason and timestamp, plus a `Resume` button.
  - The pane reflects every `control-update` SSE event live.
- Live list: cards per session; left border coloured by status (PLANNING = muted, EXECUTING = blue, COMMITTED = green, FAILED = red, HALTED = orange, STUCK = pale red).
  - Header row: prompt (first 80 chars), status pill, action count, elapsed time.
  - Click to expand:
    - **Plan ledger**: target files (yellow list), steps (numbered list), currentDispatch (action-kind pill + targetFile + instruction).
    - **Edit log**: vertical timeline of `EditEntry` rows. Each row shows attempt number, action-kind pill (`READ_FILE` muted, `EDIT_FILE` blue, `RUN_TESTS` purple), target file, verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `CI_GATE_FAILED` orange, `FAILED` red, `UNSAFE` dark red), and a collapsed `<pre>` of `content` / `diff` / `testOutput` (click to expand).
    - **Commit summary** (only when status is COMMITTED): commit message + filesChanged list + testsPassed badge.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
