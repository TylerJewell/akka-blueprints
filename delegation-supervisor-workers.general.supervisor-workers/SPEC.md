# SPEC — supervisor-workers

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Multi-Agent Collaboration (Supervisor).
**One-line pitch:** Submit a task; a supervisor classifies and routes it to specialist workers in parallel, then synthesises their outputs into a unified response.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to specialist AutonomousAgents based on the supervisor's routing decision, gathers their results, and asks the supervisor to synthesise a unified response. The blueprint also demonstrates a **before-tool-call guardrail** that constrains what each worker is permitted to invoke, and an **eval-event** that scores completed tasks for quality assurance.

## 3. User-facing flows

The user opens the App UI tab and submits a task description via the form.

1. The system creates a `CollaborationTask` record in `ROUTING` and starts a `CollaborationWorkflow`.
2. The supervisor classifies the task and emits a `RoutingDecision { researchQuery, chartDescription }` specifying what each worker should do.
3. The workflow forks: `ResearchWorker` and `ChartWorker` run concurrently. Each returns a typed payload.
4. The supervisor merges the two payloads into a `TaskResult { summary, researchOutput, chartData, guardrailVerdict }`.
5. A before-tool-call guardrail runs on the supervisor's synthesis step; if the supervisor attempts a disallowed tool call, the task moves to `BLOCKED`. Otherwise, the task moves to `COMPLETED`.
6. If either worker times out after 60 seconds, the workflow short-circuits: it asks the supervisor to synthesise from whichever side returned, and the task enters `DEGRADED`.

A `TaskSimulator` (TimedAction) drips a sample task every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskSupervisor` | `AutonomousAgent` | Classifies the task, emits a routing decision, synthesises the merged worker outputs. | `CollaborationWorkflow` | returns typed result to workflow |
| `ResearchWorker` | `AutonomousAgent` | Runs web-search-style lookups for the assigned query. Seeded "search tool" returns canned results. | `CollaborationWorkflow` | — |
| `ChartWorker` | `AutonomousAgent` | Generates structured chart data from a description. Seeded "chart tool" returns canned chart payloads. | `CollaborationWorkflow` | — |
| `CollaborationWorkflow` | `Workflow` | Sequences the routing step, the parallel worker fan-out, the synthesis step, and the guardrail step. | `CollaborationEndpoint`, `TaskRequestConsumer` | `TaskEntity` |
| `TaskEntity` | `EventSourcedEntity` | Holds the task lifecycle (routing → in-progress → completed / degraded / blocked). | `CollaborationWorkflow` | `TaskView` |
| `TaskRequestQueue` | `EventSourcedEntity` | Logs each submitted task for replay and audit. | `CollaborationEndpoint`, `TaskSimulator` | `TaskRequestConsumer` |
| `TaskView` | `View` | List-of-tasks read model. | `TaskEntity` events | `CollaborationEndpoint` |
| `TaskRequestConsumer` | `Consumer` | Listens to `TaskRequestQueue` events and starts a workflow per submission. | `TaskRequestQueue` events | `CollaborationWorkflow` |
| `TaskSimulator` | `TimedAction` | Drips a sample task every 60 s. | scheduler | `TaskRequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one completed task every 5 minutes for eval scoring; emits a `TaskEvalScored` event. | scheduler | `TaskEntity` |
| `CollaborationEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE. | — | `TaskView`, `TaskRequestQueue`, `TaskEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TaskRequest(String description, String requestedBy) {}

record RoutingDecision(String researchQuery, String chartDescription) {}

record ResearchOutput(List<ResearchFinding> findings, Instant gatheredAt) {}
record ResearchFinding(String headline, String source, String content) {}

record ChartData(String chartType, String title, List<ChartSeries> series, Instant generatedAt) {}
record ChartSeries(String label, List<Double> values) {}

record TaskResult(
    String summary,
    ResearchOutput researchOutput,
    ChartData chartData,
    String guardrailVerdict,
    Instant completedAt
) {}

record CollaborationTask(
    String taskId,
    String description,
    TaskStatus status,
    Optional<RoutingDecision> routingDecision,
    Optional<ResearchOutput> researchOutput,
    Optional<ChartData> chartData,
    Optional<TaskResult> result,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus { ROUTING, IN_PROGRESS, COMPLETED, DEGRADED, BLOCKED }
```

### Events (on `TaskEntity`)

`TaskCreated`, `TaskRouted`, `ResearchOutputAttached`, `ChartDataAttached`, `TaskCompleted`, `TaskDegraded`, `TaskBlocked`, `TaskEvalScored`.

### Events (on `TaskRequestQueue`)

`TaskSubmitted { taskId, description, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ description }` → `{ taskId }`. Starts a workflow.
- `GET /api/tasks` — list all tasks. Optional `?status=ROUTING|IN_PROGRESS|COMPLETED|DEGRADED|BLOCKED`.
- `GET /api/tasks/{id}` — one task.
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Multi-Agent Collaboration (Supervisor)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task description, live list of tasks with status pills, expand-row to see research output + chart data + synthesised summary + eval score.

Browser title: `<title>Akka Sample: Multi-Agent Collaboration (Supervisor)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `TaskSupervisor`): intercepts each tool call made by the supervisor and blocks any call outside the permitted tool list. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one completed task every 5 minutes and emits a `TaskEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `TaskSupervisor` → `prompts/supervisor.md`. Routes the task to workers; later synthesises results.
- `ResearchWorker` → `prompts/research-worker.md`. Runs search-style lookups; returns `ResearchOutput`.
- `ChartWorker` → `prompts/chart-worker.md`. Generates chart data; returns `ChartData`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a task; it progresses ROUTING → IN_PROGRESS → COMPLETED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `ResearchWorker` timeout to 1 s); task enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a disallowed tool call in the supervisor; guardrail fires; task enters BLOCKED.
4. **J4** — Wait after a successful completion; the task's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named supervisor-workers demonstrating the
delegation-supervisor-workers × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-general-supervisor-workers.
Java package io.akka.samples.multiagentcollaborationsupervisor. Akka 3.6.0.
HTTP port 9925.

Components to wire (exactly):
- 3 AutonomousAgents:
  * TaskSupervisor — definition() with capability(TaskAcceptance.of(ROUTE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/supervisor.md. Returns RoutingDecision{researchQuery, chartDescription} for ROUTE
    and TaskResult{summary, researchOutput, chartData, guardrailVerdict, completedAt} for SYNTHESISE.
    A before-tool-call guardrail runs on this agent; any tool call not in the permitted list
    (SEARCH, CHART_GENERATE) causes the guardrail to return a blocked verdict immediately.
  * ResearchWorker — capability(TaskAcceptance.of(RESEARCH).maxIterationsPerTask(3)). System prompt
    from prompts/research-worker.md. Returns ResearchOutput{findings: List<ResearchFinding{headline,
    source, content}>, gatheredAt}.
  * ChartWorker — capability(TaskAcceptance.of(CHART).maxIterationsPerTask(2)). System prompt from
    prompts/chart-worker.md. Returns ChartData{chartType, title, series: List<ChartSeries{label,
    values: List<Double>}>, generatedAt}.

- 1 Workflow CollaborationWorkflow with steps:
  routeStep -> [parallel] researchStep, chartStep -> joinStep -> synthesiseStep -> guardrailStep -> emitStep.
  routeStep calls forAutonomousAgent(TaskSupervisor.class, ROUTE).
  researchStep and chartStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(CollaborationWorkflow::researchStep, ofSeconds(60)) and
  stepTimeout(CollaborationWorkflow::chartStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls synthesiseStep with whichever side returned, then ends with TaskDegraded.
  synthesiseStep calls forAutonomousAgent(TaskSupervisor.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. guardrailStep enforces the before-tool-call
  policy: if the supervisor's synthesis attempted any disallowed tool call, end with TaskBlocked.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity TaskEntity holding state CollaborationTask{taskId, description,
  TaskStatus, Optional<RoutingDecision> routingDecision, Optional<ResearchOutput> researchOutput,
  Optional<ChartData> chartData, Optional<TaskResult> result, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. TaskStatus enum: ROUTING, IN_PROGRESS, COMPLETED,
  DEGRADED, BLOCKED. Events: TaskCreated, TaskRouted, ResearchOutputAttached, ChartDataAttached,
  TaskCompleted, TaskDegraded, TaskBlocked, TaskEvalScored. Commands: createTask, routeTask,
  attachResearch, attachChart, completeTask, degradeTask, blockTask, recordEval, getTask.
  emptyState() returns CollaborationTask.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity TaskRequestQueue with command enqueueTask(description, requestedBy) emitting
  TaskSubmitted{taskId, description, requestedBy, submittedAt}.

- 1 View TaskView with row type CollaborationTaskRow (mirrors CollaborationTask minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  TaskEntity events. ONE query getAllTasks SELECT * AS tasks FROM task_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer TaskRequestConsumer subscribed to TaskRequestQueue events; on TaskSubmitted
  starts a CollaborationWorkflow with the taskId as the workflow id.

- 2 TimedActions:
  * TaskSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-tasks.jsonl and calls TaskRequestQueue.enqueueTask.
  * EvalSampler — every 5 minutes, queries TaskView.getAllTasks, picks the oldest
    COMPLETED task without an evalScore, runs a 1–5 rubric judge over the task result,
    then calls TaskEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * CollaborationEndpoint at /api with POST /tasks, GET /tasks, GET /tasks/{id},
    GET /tasks/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- WorkerTasks.java declaring four Task<R> constants: ROUTE (RoutingDecision), RESEARCH
  (ResearchOutput), CHART (ChartData), SYNTHESISE (TaskResult).
- Domain records RoutingDecision, ResearchFinding, ResearchOutput, ChartSeries, ChartData, TaskResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9925 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-tasks.jsonl with 8 canned task description lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 before-tool-call guardrail,
  E1 eval-event on-decision-eval) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = task-delegation,
  decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/supervisor.md, prompts/research-worker.md, prompts/chart-worker.md loaded at agent
  startup as system prompts.
- README.md at the project root: title "Akka Sample: Multi-Agent Collaboration (Supervisor)",
  one-line pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Multi-Agent Collaboration (Supervisor)</title>.
  No subtitle on the Overview tab.

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
  research-worker.json, chart-worker.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    supervisor.json — list of either RoutingDecision or TaskResult objects.
      4–6 RoutingDecision entries (researchQuery + chartDescription pairs) and
      4–6 TaskResult entries (each with a 60–120 word summary, a researchOutput
      with 3–5 findings, a chartData with a bar or line chart with 2–3 series,
      guardrailVerdict = "ok").
    research-worker.json — 4–6 ResearchOutput entries, each with 3–5 findings
      whose source values are believable (e.g., "OECD AI Policy Observatory 2024",
      "IEEE Spectrum", "unsourced — knowledge").
    chart-worker.json — 4–6 ChartData entries, mixing "bar" and "line" chartType,
      each with a descriptive title and 2–3 ChartSeries with 4–6 numeric values.
- A MockModelProvider.seedFor(taskId) helper makes the selection
  deterministic per task id so the same task produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion WorkerTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9925 in application.conf.
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
