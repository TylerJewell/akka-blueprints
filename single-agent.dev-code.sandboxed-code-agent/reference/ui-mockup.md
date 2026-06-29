# UI mockup — sandboxed-code-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Sandboxed Code Execution</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Sandboxed Code <span class="accent">Execution</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded task (data analysis / file transformation / web screenshot) or type your own.
  3. Select a sandbox back-end (Docker is the default; requires no API key).
  4. Click **Run task** and watch the card transition through SCREENING → APPROVED → RUNNING → COMPLETED.
- Card **How it works**: one paragraph on submit → screen → execute → record; one paragraph on the two governance mechanisms (before-tool-call guardrail and automatic safety halt).
- Card **Components**: rows per component (ExecutionEntity, SandboxRouter, SafetyHaltMonitor, ExecutionWorkflow, CodeExecutionAgent, CodeScreeningGuardrail, ExecutionView, ExecutionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 Consumers, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = fully-autonomous` and `oversight.human_in_loop = false` declarations are prominent and highlighted as high-attention fields. `sandbox_network_isolation_enforced = true` and `sandbox_filesystem_isolation_enforced = true` are shown in the Compliance section. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 orange (halt).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Watch the sandbox run it.</span>`. Subtitle: `One agent, a code screener, and a resource monitor.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `Task description` textarea (with a "Load seeded task" link that picks from three seeded examples), `Sandbox back-end` dropdown (Docker / E2B / Modal / Playwright), `Wall-clock budget (s)` number input (default 30), `CPU budget (s)` number input (default 10), `Submitted by` text input, and a yellow **Run task** button.
    - Live list below: one card per execution, newest-first. Each card shows status pill, outcome badge (when complete), task description (truncated to 60 chars), sandbox back-end chip, age.
  - **Right column** — Selected-execution detail.
    - Header: status pill + outcome badge + `taskDescription`.
    - Sandbox config row: backend chip, wall-clock budget, CPU budget.
    - Generated code block: a monospace pre block with the Python source. If the code was blocked on a previous attempt, show a `BLOCKED: <pattern-name>` chip above, then the approved code below.
    - Screening outcome row: APPROVED chip (green) or BLOCKED chip (red) with the violated pattern name.
    - Output section: stdout in a monospace block, stderr (collapsed unless non-empty), exit code badge, wall-clock ms, CPU ms.
    - Generated files: a list of filenames (if any).
    - Halt section (visible only when status = HALTED): limit breached chip, measured vs. budget values.
- Status pill colours: SUBMITTED=muted, SCREENING=yellow, APPROVED=blue, RUNNING=yellow, COMPLETED=green, HALTED=orange, FAILED=red.
- Outcome badge colours: COMPLETED=green, HALTED=orange, FAILED=red.
- Back-end chip colours: DOCKER=blue, E2B=purple, MODAL=indigo, PLAYWRIGHT=teal.
- Screening chip colours: APPROVED=green, BLOCKED=red.

The generated code is displayed for transparency — the UI shows the operator exactly what the sandbox ran. The `violatedPattern` field from a blocked attempt persists in the entity's `screening` field so the operator can see that the guardrail fired, even after the agent later produced an approved version.
