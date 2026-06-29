# SPEC — autonomous-agent-pattern

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AutonomousAgent.
**One-line pitch:** A user submits a GitHub issue; one AI coding agent reads the issue (passed as a task attachment, never as inline prompt text), plans a fix, calls tools in a loop (read file, write file, run shell command, search codebase), and returns a structured `TaskOutcome` — patch diff, confidence score, and a per-step tool trace — pending a human-checkpoint approval before any permanent write lands.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the software-development domain. One `CodingAgent` (AutonomousAgent) carries the entire planning and execution decision; the surrounding components only prepare its environment, gate its actions, and audit its trace. Four governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs before every tool invocation — so shell commands are validated against an allowlist, file writes are restricted to the declared workspace path, and destructive operations are blocked before any side effect occurs.
- A **halt mechanism** (operator/regulator stop) gives any privileged caller the ability to set a `halt` flag on `AgentTaskEntity`. The agent loop checks this flag at the start of each iteration and transitions to `HALTED` if it is set.
- A **human-in-the-loop checkpoint** pauses the workflow at `checkpointStep` after planning but before writing. A human reviewer reads the proposed plan and either approves or rejects it; the loop only continues on approval.
- A **deployer-runtime monitor** (`RuntimeMonitor` Consumer) subscribes to every `ToolCallExecuted` event and runs anomaly rules non-blocking: rate spikes (more than N tool calls per minute), repeated failures, and shell commands that match alert patterns. Emits `MonitorAlert` events without halting the loop.

The blueprint shows that autonomous, open-ended tool use does not mean ungoverned — four independent controls sit around the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user enters a **GitHub issue URL** (e.g., `https://github.com/acme/payments/issues/42`) into the issue input and clicks **Submit**.
2. The UI POSTs to `/api/tasks` and receives a `taskId`. The card appears in the live list in `SUBMITTED` state.
3. Within ~1 s, the workflow's `fetchIssueStep` fetches the issue body from the GitHub API and attaches it. The card transitions to `ISSUE_FETCHED`.
4. Within ~5 s, the agent's first iteration completes the planning phase: it reads relevant files, proposes a patch outline, and transitions the card to `PLAN_READY`.
5. The **Checkpoint** panel appears in the right pane. The human reviewer reads the proposed plan (file list to change, summary of intent) and clicks **Approve** or **Reject**.
6. On approval, the workflow resumes `executeStep`. The agent calls file-write and shell tools in a loop (bounded by `maxIterationsPerTask`). Each tool call fires the before-tool-call guardrail. The card shows a live tool-call trace panel, updating via SSE.
7. Within ~60 s (real LLM) or ~2 s (mock), the agent returns `TaskOutcome`. The card transitions to `PATCH_READY`. The right pane shows the patch diff, confidence score (0–1), and the full tool-call trace.
8. The user can download the patch or dismiss the card. The live list keeps all history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AgentTaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, list, get, approve/reject checkpoint, halt, SSE; serves `/api/metadata/*`. | — | `AgentTaskEntity`, `AgentTaskView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AgentTaskEntity` | `EventSourcedEntity` | Per-task lifecycle: submitted → issue_fetched → plan_ready → checkpoint_pending → executing → patch_ready → halted → failed. Source of truth. | `AgentTaskEndpoint`, `AgentTaskWorkflow`, `RuntimeMonitor` | `AgentTaskView` |
| `AgentTaskWorkflow` | `Workflow` | One workflow per task. Steps: `fetchIssueStep` → `planStep` → `checkpointStep` → `executeStep` → `outcomeStep`. | started by `AgentTaskEndpoint` after submit | `CodingAgent`, `AgentTaskEntity` |
| `CodingAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the issue body as a task attachment; uses tools (readFile, writeFile, runShell, searchCodebase) in a loop. Returns `TaskOutcome`. | invoked by `AgentTaskWorkflow` | returns outcome |
| `ToolCallGuardrail` | — (before-tool-call hook) | Validates every tool call before execution: allowlist check, path confinement, rate check, shell-command pattern filter. | registered on `CodingAgent` | blocks or passes each tool call |
| `RuntimeMonitor` | `Consumer` | Subscribes to `ToolCallExecuted` events; detects anomalous patterns; emits `MonitorAlert` to `AgentTaskEntity`. | `AgentTaskEntity` events | `AgentTaskEntity` |
| `AgentTaskView` | `View` | Read model: one row per task for the UI. | `AgentTaskEntity` events | `AgentTaskEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record IssueRef(
    String issueUrl,
    String repoOwner,
    String repoName,
    int issueNumber,
    String submittedBy,
    Instant submittedAt
) {}

record IssueBody(
    String title,
    String body,
    List<String> labels,
    String fetchedAt
) {}

record ToolCall(
    String toolCallId,
    String toolName,        // "readFile" | "writeFile" | "runShell" | "searchCodebase"
    Map<String, String> parameters,
    ToolCallStatus status,  // ALLOWED | BLOCKED | SUCCEEDED | FAILED
    Optional<String> result,
    Optional<String> blockReason,
    Instant calledAt
) {}
enum ToolCallStatus { ALLOWED, BLOCKED, SUCCEEDED, FAILED }

record AgentPlan(
    String summary,
    List<String> filesToChange,
    String approach,
    Instant plannedAt
) {}

record PatchDiff(
    String unifiedDiff,
    List<String> filesModified,
    int linesAdded,
    int linesRemoved
) {}

record TaskOutcome(
    OutcomeStatus status,   // SUCCEEDED | PARTIAL | FAILED
    Optional<PatchDiff> patch,
    double confidenceScore,  // 0.0–1.0
    List<ToolCall> toolTrace,
    String summary,
    Instant completedAt
) {}
enum OutcomeStatus { SUCCEEDED, PARTIAL, FAILED }

record MonitorAlert(
    String alertId,
    AlertKind kind,         // RATE_SPIKE | REPEATED_FAILURE | SHELL_PATTERN
    String detail,
    Instant alertedAt
) {}
enum AlertKind { RATE_SPIKE, REPEATED_FAILURE, SHELL_PATTERN }

record AgentTask(
    String taskId,
    Optional<IssueRef> issueRef,
    Optional<IssueBody> issueBody,
    Optional<AgentPlan> plan,
    Optional<TaskOutcome> outcome,
    List<ToolCall> toolTrace,
    List<MonitorAlert> alerts,
    AgentTaskStatus status,
    boolean haltRequested,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AgentTaskStatus {
    SUBMITTED, ISSUE_FETCHED, PLAN_READY, CHECKPOINT_PENDING,
    EXECUTING, PATCH_READY, HALTED, FAILED
}
```

Events on `AgentTaskEntity`: `TaskSubmitted`, `IssueFetched`, `PlanProposed`, `CheckpointPending`, `CheckpointApproved`, `CheckpointRejected`, `ExecutionStarted`, `ToolCallExecuted`, `OutcomeRecorded`, `HaltRequested`, `TaskHalted`, `TaskFailed`, `MonitorAlertRaised`.

Every nullable lifecycle field on the `AgentTask` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/tasks` — body `{ issueUrl, submittedBy }` → `{ taskId }`.
- `GET /api/tasks` — list all tasks, newest-first.
- `GET /api/tasks/{id}` — one task.
- `POST /api/tasks/{id}/approve` — approve the checkpoint; resumes `executeStep`.
- `POST /api/tasks/{id}/reject` — reject the checkpoint; transitions entity to `FAILED`.
- `POST /api/tasks/{id}/halt` — set `haltRequested = true`; the loop halts at next iteration.
- `GET /api/tasks/sse` — Server-Sent Events; one event per state transition or tool-call execution.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Autonomous Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted tasks (status pill + outcome badge + age) and a right pane with the selected task's detail — issue body, plan summary, checkpoint panel (approve/reject), live tool-call trace, patch diff viewer, confidence score bar, and monitor-alert chips.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`): runs before every tool invocation inside `CodingAgent`. Checks: (1) the tool name is in the declared allowlist `{readFile, writeFile, runShell, searchCodebase}`; (2) any `path` parameter is confined to the declared workspace prefix; (3) any `command` parameter does not match shell-danger patterns (`rm -rf`, `curl | sh`, `chmod 777`, `sudo`, pipe-to-shell constructs); (4) the per-minute tool-call rate for this `taskId` has not exceeded the configured cap. On rejection, returns a structured block reason to the agent loop so the agent can propose an alternative. Does not count toward `maxIterationsPerTask`.
- **H1 — operator halt** (`halt`, `operator-regulator-stop`): `POST /api/tasks/{id}/halt` writes `HaltRequested` onto `AgentTaskEntity`, setting `haltRequested = true`. On each iteration start the agent loop checks `AgentTaskEntity.getTask().haltRequested()`; if true, the loop exits and the entity transitions to `HALTED` via `TaskHalted` event. The check is in the workflow's per-iteration prologue, before any tool call.
- **C1 — human-in-the-loop checkpoint** (`hitl`, `application`): after `planStep` the workflow emits `CheckpointPending` and transitions to `checkpointStep`, which pauses (up to its stepTimeout) waiting for `AgentTaskEntity` to record either `CheckpointApproved` or `CheckpointRejected`. Approval continues to `executeStep`. Rejection emits `TaskFailed` with reason `checkpoint-rejected`. No plan is executed without explicit human approval.
- **M1 — deployer-runtime monitor** (`hotl`, `deployer-runtime-monitoring`): `RuntimeMonitor` Consumer subscribes to every `ToolCallExecuted` event and runs three non-blocking anomaly rules: rate-spike (>10 tool calls in 60 s per task), repeated-failure (≥3 consecutive `FAILED` tool calls), shell-pattern (command matches alert-pattern list). On a match, calls `AgentTaskEntity.raiseMonitorAlert(alert)`. Monitor alerts appear on the UI's alert chip strip and do not halt the loop — they surface the signal for the deployer to act on.

## 9. Agent prompts

- `CodingAgent` → `prompts/coding-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached issue, plan a minimal fix, call tools to gather context, propose a patch, and return one `TaskOutcome`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a seeded GitHub issue; within 60 s the patch appears with a confidence score and a per-step tool trace.
2. **J2** — The agent attempts `runShell` with command `rm -rf /tmp/work`; the before-tool-call guardrail blocks it; the agent receives the rejection, proposes `rm /tmp/work/scratch.txt` instead, and the loop continues.
3. **J3** — Operator calls `POST /api/tasks/{id}/halt` mid-execution; the entity transitions to `HALTED` within one iteration timeout; the UI shows the partial tool trace.
4. **J4** — Human checkpoint rejects the proposed plan; the entity transitions to `FAILED` with reason `checkpoint-rejected`; no file writes occur.
5. **J5** — `RuntimeMonitor` detects a rate spike (12 tool calls in under 60 s); a `MonitorAlert` of kind `RATE_SPIKE` appears on the task's alert chip strip in the UI without halting the loop.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named autonomous-agent-pattern demonstrating the single-agent × dev-code cell.
Requires a GitHub API token (GITHUB_TOKEN env var). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-autonomous-agent-pattern. Java package io.akka.samples.autonomousagent.
Akka 3.6.0. HTTP port 9758.

Components to wire (exactly):

- 1 AutonomousAgent CodingAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/coding-agent.md>) and
  .capability(TaskAcceptance.of(RESOLVE_ISSUE).maxIterationsPerTask(12)). The task receives
  the issue body as its task ATTACHMENT (NOT as inline prompt text —
  TaskDef.attachment("issue.md", issueMarkdown.getBytes()) is the canonical call). The agent
  is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via
  the agent's guardrail-configuration block. Tools declared on the agent: readFile(path),
  writeFile(path, content), runShell(command), searchCodebase(query). Output: TaskOutcome.

- 1 Workflow AgentTaskWorkflow per taskId with five steps:
  * fetchIssueStep — calls GitHub REST API GET /repos/{owner}/{repo}/issues/{number} using
    GITHUB_TOKEN from env. Parses IssueBody. Calls AgentTaskEntity.recordIssueFetched(issueBody).
    WorkflowSettings.stepTimeout 10s.
  * planStep — emits ExecutionStarted, then calls componentClient.forAutonomousAgent(
    CodingAgent.class, "planner-" + taskId).runSingleTask(
      TaskDef.instructions("Phase: PLAN — produce an AgentPlan only, no writes.")
        .attachment("issue.md", issueBody.toMarkdown().getBytes())
    ) — returns a taskId; forTask(planTaskId).result(PLAN_ISSUE) to fetch AgentPlan.
    Calls AgentTaskEntity.recordPlan(plan). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(1).failoverTo(AgentTaskWorkflow::error).
  * checkpointStep — emits CheckpointPending. Polls AgentTaskEntity.getTask every 2s for
    CheckpointApproved or CheckpointRejected (up to 300s). On approved advance to executeStep.
    On rejected call AgentTaskEntity.fail("checkpoint-rejected"). WorkflowSettings.stepTimeout
    305s (human has up to 5 minutes to review).
  * executeStep — calls componentClient.forAutonomousAgent(
    CodingAgent.class, "executor-" + taskId).runSingleTask(
      TaskDef.instructions("Phase: EXECUTE — implement the approved plan. Use tools.")
        .attachment("issue.md", issueBody.toMarkdown().getBytes())
        .attachment("plan.md", plan.toMarkdown().getBytes())
    ) — returns TaskOutcome. Checks haltRequested at iteration start via AgentTaskEntity.
    WorkflowSettings.stepTimeout 300s with defaultStepRecovery maxRetries(1)
    .failoverTo(AgentTaskWorkflow::error).
  * outcomeStep — calls AgentTaskEntity.recordOutcome(outcome).
    WorkflowSettings.stepTimeout 5s. error step calls AgentTaskEntity.fail(reason).

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity AgentTaskEntity (one per taskId). State AgentTask{taskId: String,
  issueRef: Optional<IssueRef>, issueBody: Optional<IssueBody>, plan: Optional<AgentPlan>,
  outcome: Optional<TaskOutcome>, toolTrace: List<ToolCall>, alerts: List<MonitorAlert>,
  status: AgentTaskStatus, haltRequested: boolean, createdAt: Instant,
  finishedAt: Optional<Instant>}. AgentTaskStatus enum: SUBMITTED, ISSUE_FETCHED, PLAN_READY,
  CHECKPOINT_PENDING, EXECUTING, PATCH_READY, HALTED, FAILED. Events: TaskSubmitted{issueRef},
  IssueFetched{issueBody}, PlanProposed{plan}, CheckpointPending{}, CheckpointApproved{},
  CheckpointRejected{}, ExecutionStarted{}, ToolCallExecuted{toolCall}, OutcomeRecorded{outcome},
  HaltRequested{}, TaskHalted{}, TaskFailed{reason}, MonitorAlertRaised{alert}.
  Commands: submit, recordIssueFetched, recordPlan, markCheckpointPending, approveCheckpoint,
  rejectCheckpoint, markExecuting, recordToolCall, recordOutcome, requestHalt, haltTask,
  raiseMonitorAlert, fail, getTask. emptyState() returns AgentTask.initial("") with all
  Optional fields as Optional.empty() and toolTrace/alerts as empty lists (Lesson 3).

- 1 Consumer RuntimeMonitor subscribed to AgentTaskEntity events; on ToolCallExecuted runs
  three non-blocking anomaly rules: (1) rate-spike: count ToolCallExecuted events for this
  taskId in last 60s window; if >10 emit MonitorAlert{RATE_SPIKE}; (2) repeated-failure:
  if last 3 consecutive ToolCallExecuted events for this taskId have status FAILED emit
  MonitorAlert{REPEATED_FAILURE}; (3) shell-pattern: if toolCall.toolName=="runShell" and
  command matches alert-pattern list (/etc/passwd, /proc, /sys) emit MonitorAlert{SHELL_PATTERN}.
  On alert: calls AgentTaskEntity.raiseMonitorAlert(alert).

- 1 View AgentTaskView with row type AgentTaskRow (mirrors AgentTask). Table updater consumes
  AgentTaskEntity events. ONE query getAllTasks: SELECT * AS tasks FROM agent_task_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * AgentTaskEndpoint at /api with POST /tasks (body {issueUrl, submittedBy}; mints taskId;
    parses issueUrl into IssueRef; calls AgentTaskEntity.submit; starts AgentTaskWorkflow;
    returns {taskId}), GET /tasks (list from getAllTasks, sorted newest-first), GET /tasks/{id}
    (one row), POST /tasks/{id}/approve (calls AgentTaskEntity.approveCheckpoint), POST
    /tasks/{id}/reject (calls AgentTaskEntity.rejectCheckpoint), POST /tasks/{id}/halt (calls
    AgentTaskEntity.requestHalt), GET /tasks/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AgentTasks.java declaring two Task<R> constants:
    PLAN_ISSUE = Task.name("Plan issue fix")
      .description("Read the attached issue and produce an AgentPlan — file list and approach, no writes.")
      .resultConformsTo(AgentPlan.class)
    RESOLVE_ISSUE = Task.name("Resolve issue")
      .description("Implement the approved plan using readFile, writeFile, runShell, searchCodebase tools and return a TaskOutcome.")
      .resultConformsTo(TaskOutcome.class)
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records IssueRef, IssueBody, ToolCall, ToolCallStatus, AgentPlan, PatchDiff,
  TaskOutcome, OutcomeStatus, MonitorAlert, AlertKind, AgentTask, AgentTaskStatus.

- ToolCallGuardrail.java implementing the before-tool-call hook. Receives the proposed
  ToolCallRequest (toolName, parameters map). Runs the four checks listed in eval-matrix.yaml
  G1: allowlist, path confinement, shell-danger pattern, per-minute rate cap. On any failure
  returns Guardrail.reject(<structured block reason>). On pass returns Guardrail.allow().

- HaltChecker.java — thin helper called by AgentTaskWorkflow at the start of each execute
  iteration. Calls AgentTaskEntity.getTask().haltRequested(). Returns boolean. If true,
  the workflow calls AgentTaskEntity.haltTask() and transitions to the error step.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9758 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The CodingAgent.definition() binds the
  configured provider via the per-agent override pattern from the akka-context docs.
  Also reads ${?GITHUB_TOKEN} for the GitHub API calls in fetchIssueStep.

- src/main/resources/sample-events/seed-issues.jsonl with 3 seeded GitHub-style issues:
  (1) a null-pointer bug in a payment calculator, (2) an off-by-one in a date-range filter,
  (3) a missing input-validation check in a REST endpoint. Each issue contains enough context
  (stack trace, failing test, relevant file names) for the agent to plan a fix without
  external repository access.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 4 controls (G1, H1, C1, M1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.capabilities including code-execution = true,
  file-system-write = true; decisions.authority_level = supervised-autonomous;
  oversight.human_in_loop = true; oversight.human_on_loop = true; failure.failure_modes
  including "unconfined-shell-execution", "plan-hallucination", "runaway-tool-loop",
  "partial-patch-commit"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/coding-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Autonomous Agent", prerequisites
  (including GITHUB_TOKEN), generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of task cards; right = selected-task detail with issue body, plan summary,
  checkpoint panel, live tool-call trace, patch diff viewer, confidence score bar, and
  monitor-alert chip strip).
  Browser title exactly: <title>Akka Sample: Autonomous Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- Also inspect for GITHUB_TOKEN. If absent, ask the user how to source it, using the same
  five-option menu as for the model-provider key (mock / env var / env file / secrets URI /
  type once). NEVER write the token value to disk.
- If no model-provider key is set, ask the user how to source it, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        TaskOutcome and AgentPlan shapes. Mock tool calls succeed or fail per a seeded schedule.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with dispatch on Task<R> id.
  PLAN_ISSUE mock: 3 AgentPlan entries covering the three seeded issues. Each plan lists 1–3
    files to change and a 2-sentence approach. Plus 1 entry that returns a valid plan with
    an intentionally broad `filesToChange` list (more than 10 files) — tests that the
    checkpoint reviewer sees a suspiciously large plan.
  RESOLVE_ISSUE mock: 5 TaskOutcome entries. Three with OutcomeStatus.SUCCEEDED, one
    PARTIAL, one FAILED. Each SUCCEEDED entry includes a non-empty unifiedDiff, confidence
    ≥ 0.7, and a toolTrace with 4–8 ToolCall entries. PARTIAL has a diff but confidence < 0.5.
    FAILED has no diff. One mock RESOLVE_ISSUE entry has a ToolCall with status BLOCKED (the
    mock simulates a guardrail block so J2 is exercisable in mock mode).
  seedFor(taskId) makes per-task selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CodingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AgentTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (fetchIssueStep 10s, planStep 60s,
  checkpointStep 305s, executeStep 300s, outcomeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on AgentTask is Optional<T>. toolTrace and alerts
  are List<T> (never Optional) — they start empty and grow.
- Lesson 7: AgentTasks.java with PLAN_ISSUE and RESOLVE_ISSUE constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated names.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9758 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "GitHub API" — never T1/T2/T3/T4 in any user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (CodingAgent). The RuntimeMonitor
  is a rule-based Consumer — no LLM call. HaltChecker is a helper class — no LLM call.
- The issue body is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated fetchIssueStep uses TaskDef.attachment(...) and not string interpolation.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check after the agent returns.
- The halt check is inside the workflow's per-iteration prologue via HaltChecker, and the
  entity's haltRequested flag is the source of truth — not a JVM-level thread interrupt.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
