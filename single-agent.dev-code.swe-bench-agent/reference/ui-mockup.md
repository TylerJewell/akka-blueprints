# UI mockup — swe-bench-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SWE-Bench Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `SWE-Bench <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded task (Python AttributeError / Go nil-pointer / TypeScript type-narrowing) and load the matching snapshot with one click.
  3. Click **Submit task**.
  4. Watch the card transition through SNAPSHOT_PREPARED → PATCHING → PATCH_PRODUCED → GATE_PASSED → COMPLETED.
- Card **How it works**: one paragraph on submit → prepare → patch → gate; one paragraph on the one governance mechanism (CI test gate).
- Card **Components**: rows per component (BenchmarkTaskEntity, SnapshotPreparer, BenchmarkWorkflow, PatchEngineerAgent, PatchGuardrail, TestGateRunner, BenchmarkView, BenchmarkEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Gate runner as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `proprietary-source-code: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (C1). ID badge coloured orange (ci-gate).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a bug. <span class="accent">Get the patch.</span>`. Subtitle: `One agent, one test gate.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Task` (Python AttributeError / Go nil-pointer / TypeScript type-narrowing / custom), `Repository name` text input, `Bug description` textarea (with a "Load seeded example" link that fills the dropdown, name, and textarea), and a yellow `Submit task` button.
    - Live list below: one card per task, newest-first. Each card shows status pill, gate verdict badge (when gate ran), attempt counter, task title, age.
  - **Right column** — Selected-task detail.
    - Header: status pill + gate verdict badge + `taskTitle` + language chip.
    - Bug description: the submitted `description` text.
    - Snapshot file tree: a compact monospace list of `filePaths` from `PreparedSnapshot`, with test files highlighted in a distinct colour.
    - Diff viewer: a syntax-highlighted block of `unifiedDiff` with added lines in green and removed lines in red.
    - Confidence score: a horizontal bar (0–100) with the numeric value.
    - Test case table: columns test id, test file, outcome (coloured chip), failure message (italic, only when FAIL or ERROR).
    - Gate report footer: total tests, passed count, runtime, gate verdict chip.
- Status pill colours: SUBMITTED=muted, SNAPSHOT_PREPARED=blue, PATCHING=yellow, PATCH_PRODUCED=blue, GATE_PASSED=green, GATE_FAILED=red, COMPLETED=green, FAILED=red.
- Gate verdict badge colours: PASS=green, FAIL=red.
- Test outcome chip colours: PASS=green, FAIL=red, ERROR=orange.

The raw snapshot bytes are never displayed on this screen — only the file-tree summary from `PreparedSnapshot`. Reviewers who need the raw content fetch `GET /api/tasks/{id}` and inspect the `description.snapshotBytes` field from the JSON. This is intentional: the UI demonstrates that the model's input is the normalized form.
