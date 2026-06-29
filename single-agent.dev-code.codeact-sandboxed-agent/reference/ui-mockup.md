# UI mockup — codeact-sandboxed-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: CodeAct Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Code<span class="accent">Act</span> Agent`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded task (CSV-to-JSON / prime sieve / word frequency) or type a custom task description.
  3. Optionally paste context data into the input field.
  4. Click **Run task** and watch the card iterate through EXECUTING → SOLVED (or HALTED / FAILED).
- Card **How it works**: one paragraph on submit → execute → inspect → iterate → resolve; one paragraph on the three governance mechanisms (before-tool-call guardrail, automatic safety halt, secret sanitizer).
- Card **Components**: rows per component (TaskEntity, SecretSanitizer, ExecutionWorkflow, CodeActAgent, SandboxGuardrail, SafetyHaltMonitor, AcceptanceChecker, TaskView, TaskEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 HaltMonitor + 1 Checker as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `secrets-in-context-data: true` declaration in Data is filled and prominent. `decisions.authority_level = automated-with-halt` and `oversight.human_in_loop = false` are the distinctive answers for this domain. `TO_BE_COMPLETED_BY_DEPLOYER` fields are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, S1). ID badges coloured: G1 red (guardrail), H1 orange (halt), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Watch it run.</span>`. Subtitle: `One agent, three governance gates around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Task` (CSV-to-JSON / prime sieve / word frequency / custom), `Task description` textarea (auto-filled on dropdown pick; editable for custom), `Context data` textarea (optional; auto-filled for seeded tasks), `Submitted by` text input, and a yellow `Run task` button.
    - Live list below: one card per task, newest-first. Each card shows status pill, resolution badge (when resolved), iteration count chip, task description (truncated), age.
  - **Right column** — Selected-task detail.
    - Header: status pill + resolution badge + iteration count + task description.
    - Task description: full text, read-only.
    - Context data: shown if non-empty, monospace, read-only.
    - Acceptance criterion: shown below context data.
    - Execution log: accordion — one section per iteration. Each section shows: iteration number, the code snippet in a monospace block with syntax highlighting, and the sanitized output in a monospace block. Secret-redaction markers (`[REDACTED-AWS-KEY]` etc.) are rendered in an amber highlight. If the safety halt fired, the accordion shows a `HALTED` banner with the halt reason instead of the sanitized output.
    - Resolution section at bottom: resolution badge (SOLVED green / HALTED orange / FAILED red), `finalOutput` in a monospace block, wall-clock time.
- Status pill colours: SUBMITTED=muted, EXECUTING=yellow (pulsing), SOLVED=green, HALTED=orange, FAILED=red.
- Resolution badge colours: SOLVED=green, HALTED=orange, FAILED=red.
- Iteration count chip: small neutral badge showing `N iter` where N is the count at current state.

The raw execution output is never displayed in the UI — only the sanitized form. Operators who need the raw output access the entity directly via `GET /api/tasks/{id}` and read `codeHistory[n].rawOutput` (available on the entity but excluded from `TaskRow` in the view). This is intentional: the UI demonstrates that secrets are scrubbed before display, even though the audit trail retains the raw form for operators.
