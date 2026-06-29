# SPEC — supervisor-subagents

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Supervisor with Subagents.
**One-line pitch:** Submit a task; a supervisor classifies it and delegates subtasks to specialized subagents in parallel, then assembles their outputs into a single result.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern implemented with Akka's first-party primitives: a Workflow drives a supervisor AutonomousAgent to classify and route an incoming task, fans delegated subtasks out to two specialized subagents in parallel, collects their outputs, and reassembles them into a final result. The blueprint demonstrates a **before-agent-invocation guardrail** that policy-checks the supervisor's routing decision before any subagent is called, and an **eval-event** that samples the routing accuracy for ongoing quality measurement.

## 3. User-facing flows

The user opens the App UI tab and submits a task description via the form.

1. The system creates a `TaskRecord` in `RECEIVED` and starts a `TaskOrchestrationWorkflow`.
2. The TaskSupervisor classifies the task and emits a `RoutingPlan { dataQuery, summaryPrompt }`.
3. A before-agent-invocation guardrail checks whether the routing plan is policy-compliant before any subagent is called. If it fails, the task enters `BLOCKED`.
4. The workflow fans out: both subagents run concurrently. Each returns a typed payload.
5. The TaskSupervisor assembles the subagent outputs into a `TaskResult { narrative, data, routingRationale, assembledAt }`.
6. If either subagent times out after 60 seconds, the workflow short-circuits: the supervisor assembles from whichever side returned and the task enters `DEGRADED`.

A `TaskSimulator` (TimedAction) drips a sample task every 60 seconds so the App UI is non-empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskSupervisor` | `AutonomousAgent` | Classifies an incoming task, produces a routing plan, then assembles subagent outputs into a final result. | `TaskOrchestrationWorkflow` | returns typed result to workflow |
| `DataSubagent` | `AutonomousAgent` | Retrieves and formats structured data for a delegated subtask. | `TaskOrchestrationWorkflow` | — |
| `SummarySubagent` | `AutonomousAgent` | Produces a human-readable narrative from structured inputs. | `TaskOrchestrationWorkflow` | — |
| `TaskOrchestrationWorkflow` | `Workflow` | Runs routing, guardrail, parallel fan-out, and assembly steps. | `TaskEndpoint`, `TaskRequestConsumer` | `TaskEntity` |
| `TaskEntity` | `EventSourcedEntity` | Holds the task lifecycle (received → routing → in-progress → completed / degraded / blocked). | `TaskOrchestrationWorkflow` | `TaskView` |
| `TaskQueue` | `EventSourcedEntity` | Logs each submitted task for replay/audit. | `TaskEndpoint`, `TaskSimulator` | `TaskRequestConsumer` |
| `TaskView` | `View` | List-of-tasks read model. | `TaskEntity` events | `TaskEndpoint` |
| `TaskRequestConsumer` | `Consumer` | Listens to `TaskQueue` events and starts a workflow per submission. | `TaskQueue` events | `TaskOrchestrationWorkflow` |
| `TaskSimulator` | `TimedAction` | Drips a sample task every 60 s. | scheduler | `TaskQueue` |
| `RoutingEvalSampler` | `TimedAction` | Samples one completed task every 5 minutes for routing-accuracy scoring; emits a `RoutingEvalScored` event. | scheduler | `TaskEntity` |
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE. | — | `TaskView`, `TaskQueue`, `TaskEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TaskRequest(String description, String submittedBy) {}

record RoutingPlan(String dataQuery, String summaryPrompt) {}

record DataBundle(List<DataItem> items, Instant retrievedAt) {}
record DataItem(String label, String value, String source) {}

record SummaryOutput(String narrative, List<String> keyPoints, Instant generatedAt) {}

record TaskResult(
    String narrative,
    DataBundle data,
    SummaryOutput summary,
    String routingRationale,
    Instant assembledAt
) {}

record TaskRecord(
    String taskId,
    String description,
    TaskStatus status,
    Optional<RoutingPlan> routingPlan,
    Optional<DataBundle> data,
    Optional<SummaryOutput> summary,
    Optional<TaskResult> result,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus { RECEIVED, ROUTING, IN_PROGRESS, COMPLETED, DEGRADED, BLOCKED }
```

### Events (on `TaskEntity`)

`TaskReceived`, `TaskRouted`, `SubagentDataReturned`, `SubagentSummaryReturned`, `TaskCompleted`, `TaskDegraded`, `TaskBlocked`, `RoutingEvalScored`.

### Events (on `TaskQueue`)

`TaskSubmitted { taskId, description, submittedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ description }` → `{ taskId }`. Starts a workflow.
- `GET /api/tasks` — list all tasks. Optional `?status=RECEIVED|ROUTING|IN_PROGRESS|COMPLETED|DEGRADED|BLOCKED`.
- `GET /api/tasks/{id}` — one task.
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Supervisor with Subagents"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task description, live list of tasks with status pills, expand-row to see the data bundle, summary narrative, assembled result, and eval score.

Browser title: `<title>Akka Sample: Supervisor with Subagents</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — routing guardrail** (`before-agent-invocation` on `TaskSupervisor`): policy-checks the routing plan before any subagent is invoked. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `RoutingEvalSampler` (TimedAction) picks one completed task every 5 minutes and emits a `RoutingEvalScored` event with a 1–5 routing-accuracy score and a short rationale.

## 9. Agent prompts

- `TaskSupervisor` → `prompts/supervisor.md`. Classifies the incoming task description, emits a routing plan; later assembles subagent outputs into the final result.
- `DataSubagent` → `prompts/data-subagent.md`. Retrieves and formats structured data; returns `DataBundle`.
- `SummarySubagent` → `prompts/summary-subagent.md`. Produces a human-readable narrative; returns `SummaryOutput`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a task description; task progresses RECEIVED → ROUTING → IN_PROGRESS → COMPLETED within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a subagent timeout (set `DataSubagent` step timeout to 1 s); task enters DEGRADED with partial output.
3. **J3** — Inject a routing guardrail failure; task enters BLOCKED with a failure reason.
4. **J4** — Wait after a completed task; the task row shows a routing eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named supervisor-subagents demonstrating the
delegation-supervisor-workers × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-general-supervisor-subagents.
Java package io.akka.samples.supervisorwithsubagents. Akka 3.6.0.
HTTP port 9988.

Components to wire (exactly):
- 3 AutonomousAgents:
  * TaskSupervisor — definition() with capability(TaskAcceptance.of(ROUTE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(ASSEMBLE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/supervisor.md. Returns RoutingPlan{dataQuery, summaryPrompt} for ROUTE and
    TaskResult{narrative, data, summary, routingRationale, assembledAt} for ASSEMBLE.
  * DataSubagent — capability(TaskAcceptance.of(FETCH_DATA).maxIterationsPerTask(3)). System
    prompt from prompts/data-subagent.md. Returns DataBundle{items: List<DataItem{label, value,
    source}>, retrievedAt}.
  * SummarySubagent — capability(TaskAcceptance.of(SUMMARISE).maxIterationsPerTask(2)). System
    prompt from prompts/summary-subagent.md. Returns SummaryOutput{narrative, keyPoints:
    List<String>, generatedAt}.

- 1 Workflow TaskOrchestrationWorkflow with steps:
  routeStep -> guardrailStep -> [parallel] fetchDataStep, summariseStep
            -> assembleStep -> emitStep.
  routeStep calls forAutonomousAgent(TaskSupervisor.class, ROUTE).
  guardrailStep runs the before-agent-invocation guardrail on the RoutingPlan; on failure,
  transition to blockStep that calls TaskEntity.block and ends with TaskBlocked.
  fetchDataStep and summariseStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(TaskOrchestrationWorkflow::fetchDataStep, ofSeconds(60))
  and stepTimeout(TaskOrchestrationWorkflow::summariseStep, ofSeconds(60)). On either timeout,
  transition to degradeStep that calls assembleStep with whichever side returned, then ends with
  TaskDegraded. assembleStep calls forAutonomousAgent(TaskSupervisor.class, ASSEMBLE) with the
  merged inputs; give assembleStep a 90s stepTimeout. WorkflowSettings is nested inside Workflow
  — no import.

- 1 EventSourcedEntity TaskEntity holding state TaskRecord{taskId, description,
  TaskStatus, Optional<RoutingPlan> routingPlan, Optional<DataBundle> data,
  Optional<SummaryOutput> summary, Optional<TaskResult> result,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. TaskStatus enum: RECEIVED, ROUTING,
  IN_PROGRESS, COMPLETED, DEGRADED, BLOCKED. Events: TaskReceived, TaskRouted,
  SubagentDataReturned, SubagentSummaryReturned, TaskCompleted, TaskDegraded,
  TaskBlocked, RoutingEvalScored. Commands: createTask, routeTask, attachData,
  attachSummary, completeTask, degradeTask, blockTask, recordEval, getTask.
  emptyState() returns TaskRecord.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity TaskQueue with command submitTask(description, submittedBy) emitting
  TaskSubmitted{taskId, description, submittedBy, submittedAt}.

- 1 View TaskView with row type TaskRow (mirrors TaskRecord minus heavy nested payloads;
  every nullable field is Optional<T>). Table updater consumes TaskEntity events. ONE query
  getAllTasks SELECT * AS tasks FROM task_view. No WHERE status filter (Akka cannot
  auto-index enum columns) — caller filters client-side.

- 1 Consumer TaskRequestConsumer subscribed to TaskQueue events; on TaskSubmitted starts a
  TaskOrchestrationWorkflow with the taskId as the workflow id.

- 2 TimedActions:
  * TaskSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-tasks.jsonl and calls TaskQueue.submitTask.
  * RoutingEvalSampler — every 5 minutes, queries TaskView.getAllTasks, picks the oldest
    COMPLETED task without an evalScore, runs a 1-5 routing-accuracy judge over the
    RoutingPlan and TaskResult, then calls TaskEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks, GET /tasks, GET /tasks/{id},
    GET /tasks/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- SupervisorTasks.java declaring four Task<R> constants: ROUTE (RoutingPlan), FETCH_DATA
  (DataBundle), SUMMARISE (SummaryOutput), ASSEMBLE (TaskResult).
- Domain records RoutingPlan, DataItem, DataBundle, SummaryOutput, TaskResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9988 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-tasks.jsonl with 8 canned task description lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 routing guardrail
  before-agent-invocation, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = task-delegation,
  decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/supervisor.md, prompts/data-subagent.md, prompts/summary-subagent.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Supervisor with Subagents", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Supervisor with Subagents</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (supervisor.json,
  data-subagent.json, summary-subagent.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    supervisor.json — list of either RoutingPlan or TaskResult objects.
      4–6 RoutingPlan entries (dataQuery + summaryPrompt pairs) and
      4–6 TaskResult entries (each with a 60–120 word narrative, a 3–5 item
      data bundle, a 3–5 key-point summary, routingRationale a short sentence).
    data-subagent.json — 4–6 DataBundle entries, each with 3–5 items whose
      source values are plausible (e.g., "internal-db", "api-response", "unsourced — knowledge").
    summary-subagent.json — 4–6 SummaryOutput entries, each with a one-paragraph
      narrative and 3–5 short key-point bullets.
- A MockModelProvider.seedFor(taskId) helper makes the selection deterministic
  per task id so the same task produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s subagents,
  90s assembly); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion SupervisorTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9988 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
