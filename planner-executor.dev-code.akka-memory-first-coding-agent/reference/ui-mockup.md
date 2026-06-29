# UI mockup — akka-memory-first-coding-agent

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from Section 13 of the blueprint-authoring guide.

Browser title: `<title>Akka Sample: Code Memory-First Coding Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index.

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not acceptable.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these, state names in the `EditSessionEntity` state machine render invisible and transition labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Code Memory-First <span class="accent">Coding Agent</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — run init on the seeded project in the App UI tab, wait for memory blocks to appear, then submit an edit request and watch the patch loop run.
- Card **How it works**: one paragraph naming the research phase (research planner + code reader + memory writer) and the edit phase (edit planner + edit executor), the guardrail, and the test gate.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, two sequences, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (5 agents, 2 workflows, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, init sequence, state machine for `EditSessionEntity`, ER) with Akka theme variables.
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
- ID badges coloured per mechanism: `guardrail` red (`G1`), `ci-gate` blue (`CI1`), `halt` orange (`HO1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Research a codebase. <span class="accent">Then edit it.</span>` Subtitle: `The simulator inits a sample project on startup so the page is never empty.`

- **Project init pane**: text field labelled "Project path", `Init project` button (yellow).
- **Edit request pane** (shown only when a `READY` project is selected): text area labelled "Edit instruction", `Submit edit` button (yellow), project selector dropdown.
- **Operator controls pane** (top right):
  - If `halted=false`: yellow `Halt destructive edits` button plus a reason text field.
  - If `halted=true`: muted `HALTED` pill with reason and timestamp plus a `Resume` button.
  - The pane reflects every `control-update` SSE event live.

- **Projects list**: cards per project; left border coloured by status (RESEARCHING = blue, READY = green, FAILED = red).
  - Header row: project path, status pill, memory-block count, elapsed time.
  - Click to expand:
    - **Research plan**: files-to-read list, questions list.
    - **Memory blocks**: one card per block (name as heading, content paragraph, source-files list).
    - **System prompt**: code block showing the current rewritten prompt.

- **Sessions list**: cards per edit session; left border coloured by status (PLANNING = muted, APPLYING = blue, COMPLETED = green, TESTS_FAILED = yellow, FAILED = red, HALTED = orange, TIMED_OUT = pale red).
  - Header row: instruction (first 80 chars), status pill, patch count, elapsed time.
  - Click to expand:
    - **Patch plan**: rationale line, numbered list of `FileEdit` operations (path + kind pill).
    - **Patch log**: vertical timeline of `PatchEntry` rows. Each row shows attempt number, file path, kind pill (`INSERT` blue, `REPLACE` yellow, `DELETE` red), verdict pill (`APPLIED` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red, `TESTS_FAILED` orange), collapsed `<pre>` of `diff` (click to expand), and — when verdict is `TESTS_FAILED` — a collapsed test-output block showing failed count and output.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
