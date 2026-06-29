# UI mockup — autonomous-agent-pattern

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Autonomous Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Autonomous<span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Paste a GitHub issue URL (or load one of the three seeded examples).
  3. Click **Submit**. Watch the card reach `CHECKPOINT_PENDING`.
  4. Read the proposed plan and click **Approve** — the agent executes and the patch appears.
- Card **How it works**: one paragraph on submit → fetch → plan → checkpoint → execute → outcome; one paragraph on the four governance mechanisms (tool-call guardrail, human checkpoint, halt mechanism, runtime monitor).
- Card **Components**: rows per component (AgentTaskEntity, AgentTaskWorkflow, CodingAgent, ToolCallGuardrail, HaltChecker, RuntimeMonitor, AgentTaskView, AgentTaskEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 helper class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `code-execution: true` and `file-system-write: true` declarations in Capabilities are filled and prominent. `decisions.authority_level = supervised-autonomous` and `oversight.human_approves_plan_before_any_write = true` are the distinctive answers. `TO_BE_COMPLETED_BY_DEPLOYER` fields render in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with four rows (G1, H1, C1, M1). ID badges coloured: G1 red (guardrail), H1 orange (halt), C1 yellow (hitl), M1 blue (hotl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an issue. <span class="accent">Review the patch.</span>`. Subtitle: `One agent, four governance controls around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `GitHub Issue URL` text input (with a "Load seeded example" link that populates the field with one of three seeded issue URLs), `Submitted by` text input, and a yellow `Submit` button.
    - Live list below: one card per task, newest-first. Each card shows status pill, outcome badge (when patch landed), confidence bar (when outcome landed), repository + issue number, age. Cards with `MonitorAlert` entries show an orange alert chip.
  - **Right column** — Selected-task detail.
    - Header: status pill + outcome badge + confidence score bar + issue title.
    - **Issue body panel**: the full fetched issue text in a monospace block (title, body, labels).
    - **Plan panel**: shown when status ≥ `PLAN_READY`. Lists `filesToChange` as chips and the `approach` paragraph.
    - **Checkpoint panel**: shown when status = `CHECKPOINT_PENDING`. Displays the full `AgentPlan`. Two buttons: green `Approve` and red `Reject`. Disabled once a decision is recorded.
    - **Live tool-call trace panel**: shown when status = `EXECUTING` or later. A scrolling list of tool-call rows: tool name chip (readFile / writeFile / runShell / searchCodebase), status chip (ALLOWED / BLOCKED / SUCCEEDED / FAILED), parameter summary, result excerpt. BLOCKED rows show the `blockReason` in red italic.
    - **Patch diff panel**: shown when `outcome` is present. A syntax-highlighted unified diff. `linesAdded` in green, `linesRemoved` in red. Confidence score bar (0–1, colour: red ≤ 0.4, yellow 0.4–0.7, green > 0.7).
    - **Monitor alerts**: an alert chip strip above the patch panel. Each chip shows `AlertKind` label and timestamp. Empty strip when no alerts.
    - **Halt button**: a red `Halt` button in the panel header, active only when status = `EXECUTING`.
- Status pill colours: SUBMITTED=muted, ISSUE_FETCHED=blue, PLAN_READY=blue, CHECKPOINT_PENDING=yellow, EXECUTING=yellow, PATCH_READY=green, HALTED=orange, FAILED=red.
- Outcome badge colours: SUCCEEDED=green, PARTIAL=yellow, FAILED=red.
- Tool-call status chip colours: ALLOWED=muted, BLOCKED=red, SUCCEEDED=green, FAILED=red.
