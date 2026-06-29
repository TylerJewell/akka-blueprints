# SPEC — shared-task-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Shared Task Team.
**One-line pitch:** Submit a goal; a lead agent decomposes it into tasks on a shared board, a team of worker agents claim tasks on their own, produce result artifacts, run a quality check, and message each other when they need to coordinate.

## 2. What this blueprint demonstrates

The **team-shared-list** coordination pattern wired with Akka's first-party primitives: a planning workflow runs a goal-planner agent that writes a list of tasks onto a shared board, and a fixed roster of worker agents each runs its own durable loop that pulls open tasks off that board, claims them atomically, produces a result artifact, and marks them done. No central dispatcher hands work out — the workers self-organise around the shared list, and the board entity's single-writer guarantee is what stops two workers claiming the same task. The blueprint also demonstrates two governance mechanisms a self-organising team needs: a **before-agent-response guardrail** that vetoes synthesized team output before it reaches the user, and an **on-decision eval** that scores synthesized output for consistency across workers; plus an **operator halt** that freezes the whole team.

## 3. User-facing flows

The user opens the App UI tab and submits a goal via the form (a title plus a free-text description).

1. The system logs the submission on `IntakeQueue` and a `Consumer` starts a `PlanningWorkflow` for the goal.
2. The `GoalPlanner` agent decomposes the brief into a `TaskPlan` — a list of tasks, each with a title, a description, and the titles of tasks it depends on. The workflow writes one `TaskEntity` per task (status `OPEN`) and moves the goal to `READY`.
3. The worker roster (`worker-1`, `worker-2`, `worker-3`) is already running — each worker is a `WorkerWorkflow` that polls the shared `TaskBoardView` for an `OPEN` task whose dependencies are all `DONE`.
4. A worker claims an eligible task. The claim is an atomic command on that task's `TaskEntity`; if two workers race for the same task, one wins and the other goes back to polling.
5. The winning worker runs the `WorkerAgent`, which produces a `ResultArtifact` (a set of output sections plus a summary). Every response passes through a before-agent-response guardrail; a response that fails the safety check is refused and the task is recorded `BLOCKED`.
6. The artifact runs through a quality check. On pass, the task becomes `DONE`. On fail, the worker retries up to a fixed limit, then the task becomes `BLOCKED`.
7. If the agent reports it needs another worker's output, it raises a `PeerRequest`. The task goes `BLOCKED`, a `PeerMessage` lands in the target worker's `WorkerMailbox`, and the worker who can answer posts a reply that unblocks the waiting task.
8. When every task on a goal is `DONE`, the goal moves to `COMPLETED`. A final synthesized summary is scored by an on-decision eval for cross-worker consistency.

A `GoalSimulator` (TimedAction) drips a sample goal brief every 60 seconds so the App UI is not empty when first loaded. A `StuckTaskMonitor` (TimedAction) releases any task that has been claimed but idle for more than two minutes, returning it to `OPEN` so another worker can pick it up.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GoalPlanner` | `AutonomousAgent` | Decomposes a goal brief into a list of tasks with dependencies. | `PlanningWorkflow` | returns typed `TaskPlan` to workflow |
| `WorkerAgent` | `AutonomousAgent` | Claims and completes one task; returns a `ResultArtifact`; may raise a `PeerRequest`. Run as several instances (`worker-1`, `worker-2`, `worker-3`). | `WorkerWorkflow` | returns typed result to workflow |
| `PlanningWorkflow` | `Workflow` | Runs `GoalPlanner`, creates one `TaskEntity` per task, moves the goal to `READY`. | `GoalRequestConsumer` | `GoalPlanner`, `TaskEntity`, `GoalEntity` |
| `WorkerWorkflow` | `Workflow` | Per-worker durable loop: poll → claim → work → quality check → done/blocked/peer. | `Bootstrap` (one instance per worker id) | `TaskEntity`, `WorkerAgent`, `WorkerMailbox`, `SystemControl` |
| `GoalEntity` | `EventSourcedEntity` | Goal lifecycle: brief, task ids, status. | `PlanningWorkflow`, `GoalCompletionMonitor` | goal read via endpoint |
| `TaskEntity` | `EventSourcedEntity` | One per task. Atomic `claim`; full task lifecycle. | `PlanningWorkflow`, `WorkerWorkflow`, `StuckTaskMonitor` | `TaskBoardView` |
| `WorkerMailbox` | `EventSourcedEntity` | One per worker. Inbound peer messages and replies. | `WorkerWorkflow` | `SharedTaskEndpoint` |
| `IntakeQueue` | `EventSourcedEntity` | Single instance; logs each submitted goal for replay/audit. | `SharedTaskEndpoint`, `GoalSimulator` | `GoalRequestConsumer` |
| `SystemControl` | `KeyValueEntity` | Operator halt flag for the whole team. | `SharedTaskEndpoint` | `WorkerWorkflow`, `WorkerAgent` guardrail |
| `TaskBoardView` | `View` | The shared task list read model the workers poll and the UI streams. | `TaskEntity` events | `WorkerWorkflow`, `SharedTaskEndpoint` |
| `GoalRequestConsumer` | `Consumer` | Listens to `IntakeQueue` events; starts a `PlanningWorkflow` per submission. | `IntakeQueue` events | `PlanningWorkflow` |
| `GoalSimulator` | `TimedAction` | Drips a sample goal brief every 60 s. | scheduler | `IntakeQueue` |
| `StuckTaskMonitor` | `TimedAction` | Every 30 s, releases tasks claimed but idle > 2 min back to `OPEN`. | scheduler | `TaskEntity` |
| `SharedTaskEndpoint` | `HttpEndpoint` | `/api/*` — submit, list tasks, get goal, peer reply, halt/resume, SSE. | — | `IntakeQueue`, `TaskBoardView`, `GoalEntity`, `WorkerMailbox`, `SystemControl` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions; starts one `WorkerWorkflow` per worker id. | — | `WorkerWorkflow`, scheduler |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record GoalBrief(String goalId, String title, String description, String requestedBy) {}

record TaskSpec(String title, String description, List<String> dependsOn) {}
record TaskPlan(String planSummary, List<TaskSpec> tasks) {}

record OutputSection(String heading, String content) {}
record PeerRequest(String toWorker, String question) {}
record ResultArtifact(String taskId, List<OutputSection> sections, String summary, Optional<PeerRequest> peerRequest) {}

record QualityReport(boolean passed, List<String> failures, Instant checkedAt) {}
record PeerMessage(String messageId, String fromWorker, String toWorker, String taskId,
                   String question, Instant sentAt, Optional<String> reply, Optional<Instant> repliedAt) {}
```

### Entity state — `Task` (state of `TaskEntity`, basis of the view row)

```java
record Task(
    String taskId,
    String goalId,
    String title,
    String description,
    List<String> dependsOn,
    TaskStatus status,
    Optional<String>  claimedBy,
    Optional<Instant> claimedAt,
    Optional<String>  artifactSummary,
    Optional<Integer> sectionCount,
    Optional<QualityReport> qualityReport,
    Optional<String>  blockedReason,
    Optional<Instant> completedAt,
    Instant createdAt
) {}

enum TaskStatus { OPEN, CLAIMED, IN_PROGRESS, IN_REVIEW, DONE, BLOCKED }
```

### Entity state — `Goal` (state of `GoalEntity`)

```java
record Goal(
    String goalId,
    String title,
    String description,
    String requestedBy,
    GoalStatus status,
    List<String> taskIds,
    Optional<String> planSummary,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum GoalStatus { PLANNING, READY, IN_PROGRESS, COMPLETED }
```

### Events

`TaskEntity`: `TaskCreated`, `TaskClaimed`, `TaskStarted`, `ArtifactSubmitted`, `QualityPassed`, `QualityFailed`, `TaskBlocked`, `TaskReleased`, `TaskCompleted`.
`GoalEntity`: `GoalCreated`, `GoalPlanned`, `GoalStarted`, `GoalCompleted`.
`WorkerMailbox`: `MessagePosted`, `MessageReplied`.
`IntakeQueue`: `GoalSubmitted`.
`SystemControl` is a `KeyValueEntity` — state transitions are command-driven, no events.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/goals` — body `{ title, description, requestedBy? }` → `{ goalId }`. Logs the submission and starts planning.
- `GET /api/goals/{id}` — one goal with its task ids and status.
- `GET /api/tasks` — list all tasks. Optional `?goalId=...` and `?status=...`, filtered client-side.
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `GET /api/mailbox/{workerId}` — peer messages for a worker.
- `POST /api/mailbox/{workerId}/messages/{messageId}/reply` — body `{ reply }`. Posts a reply that unblocks the waiting task.
- `POST /api/control/halt` — body `{ reason, by }`. Sets the operator halt flag.
- `POST /api/control/resume` — clears the halt flag.
- `GET /api/control` — current halt state.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Shared Task Team</title>`.

- **Overview** — eyebrow "Overview" + headline "Shared Task Team"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, task state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a goal submission form, a live board grouped by status column (Open / Claimed / In progress / In review / Done / Blocked), per-task cards showing the claiming worker and artifact summary, a peer-mailbox panel with reply boxes, and a Halt / Resume control.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** (`before-agent-response` flavor, on `WorkerAgent`): every synthesized response from the worker agent passes through a guardrail that refuses output containing unsafe instructions, hallucinated citations, or any content produced while the team is halted. A refused response records the task `BLOCKED` with the guardrail reason. Blocking.
- **E1 — on-decision eval** (`on-decision-eval` flavor): after each goal reaches `COMPLETED`, the synthesized outputs from all contributing workers are scored for consistency — verifying that sections do not contradict each other and the plan summary is supported. Non-blocking; recorded alongside the goal entity.
- **HT1 — operator halt** (`halt`, `operator-regulator-stop` flavor): an operator can freeze the whole team through `SystemControl`; `WorkerWorkflow` stops claiming and the before-agent-response guardrail refuses every response generation until resume. System-level.

## 9. Agent prompts

- `GoalPlanner` → `prompts/goal-planner.md`. Decomposes a goal brief into a dependency-ordered task list.
- `WorkerAgent` → `prompts/worker-agent.md`. Claims a task, produces the result artifact, raises a peer request when it needs another worker's output.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a goal; the planner writes tasks to the board; workers claim and complete them; the goal reaches `COMPLETED`. The board updates live via SSE.
2. **J2** — Two workers poll at the same instant; each task ends up claimed by exactly one worker (no double-claim).
3. **J3** — A worker raises a peer request; the task goes `BLOCKED`, a message lands in the peer's mailbox, and a reply unblocks it.
4. **J4** — A worker's synthesized response fails the before-agent-response guardrail; the response is refused and the task is recorded `BLOCKED` with the guardrail reason.
5. **J5** — The operator halts the team; claiming pauses and response generation is refused; resume continues the work.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named shared-task-team demonstrating the
team-shared-list × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact team-shared-list-general-shared-task-team.
Java package io.akka.samples.teamsmultiagentcoordination. Akka 3.6.0. HTTP port 9881.

Components to wire (exactly):
- 2 AutonomousAgents:
  * GoalPlanner — definition() with capability(TaskAcceptance.of(PLAN)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/goal-planner.md.
    Returns TaskPlan{planSummary, tasks: List<TaskSpec{title, description,
    dependsOn: List<String>}>}.
  * WorkerAgent — definition() with capability(TaskAcceptance.of(WORK)
    .maxIterationsPerTask(3)). System prompt from prompts/worker-agent.md. Returns
    ResultArtifact{taskId, sections: List<OutputSection{heading, content}>, summary,
    peerRequest: Optional<PeerRequest{toWorker, question}>}. Run as several
    runtime instances addressed by instanceId (worker-1, worker-2, worker-3) — ONE
    agent class, several instance ids; never generate three agent classes.
    Register a before-agent-response guardrail on this agent (control G1).

- 1 Workflow PlanningWorkflow with steps decomposeStep -> createTasksStep ->
  readyStep. decomposeStep calls forAutonomousAgent(GoalPlanner.class, goalId)
  .runSingleTask(PLAN.instructions(brief)) then forTask(taskId)
  .result(PLAN). createTasksStep writes one TaskEntity per TaskSpec with a
  deterministic taskId = goalId + "-t" + index and calls
  GoalEntity.recordPlan(taskIds, planSummary). readyStep ends the workflow.
  Override settings() with stepTimeout(decomposeStep, 90s).

- 1 Workflow WorkerWorkflow, one durable instance per worker id. State carries
  workerId and the currently-claimed taskId (Optional). Steps:
  pollStep -> claimStep -> workStep -> qualityStep -> finishStep, with a
  self-scheduled resume timer for idling. pollStep: read SystemControl; if halted,
  schedule a 5s resume timer and pause. Otherwise query
  TaskBoardView.getAllTasks, pick the oldest OPEN task whose dependsOn titles
  are all DONE; if none, schedule a 5s resume timer and pause; else go to
  claimStep. claimStep: call TaskEntity.claim(workerId); if the entity reports
  the task is no longer OPEN (lost the race), go back to pollStep; else
  TaskEntity.start and go to workStep. workStep (stepTimeout 90s): call
  forAutonomousAgent(WorkerAgent.class, workerId).runSingleTask(
  WORK.instructions(task)). If a guardrail refusal comes back, call
  TaskEntity.block(reason) and go to pollStep. On a PeerRequest in the result,
  call WorkerMailbox(toWorker).post(message) and TaskEntity.block("waiting on
  peer: " + toWorker), then go to pollStep. Otherwise call
  TaskEntity.submitArtifact(artifact) and go to qualityStep. qualityStep: run the
  deterministic QualityChecker over the artifact; on pass call
  TaskEntity.passQuality(report) (-> DONE) and go to finishStep; on fail up to 2
  retries call TaskEntity.failQuality(report) and loop workStep; after retries
  exhausted call TaskEntity.block("quality checks failed") and go to pollStep.
  finishStep: clear the claimed task and go to pollStep.

- 4 EventSourcedEntities:
  * TaskEntity holding Task state. Commands: createTask, claim, start,
    submitArtifact, passQuality, failQuality, block, release, getTask. claim emits
    TaskClaimed ONLY when status==OPEN; otherwise it is a no-op that returns the
    current Task so the caller can detect the lost race. TaskStatus enum: OPEN,
    CLAIMED, IN_PROGRESS, IN_REVIEW, DONE, BLOCKED. Events: TaskCreated,
    TaskClaimed, TaskStarted, ArtifactSubmitted, QualityPassed, QualityFailed,
    TaskBlocked, TaskReleased, TaskCompleted. emptyState() returns
    Task.initial("", "") with NO commandContext() reference.
  * GoalEntity holding Goal state. Commands: createGoal, recordPlan, start,
    complete, getGoal. GoalStatus enum: PLANNING, READY, IN_PROGRESS, COMPLETED.
    Events: GoalCreated, GoalPlanned, GoalStarted, GoalCompleted. emptyState()
    returns Goal.initial("") with NO commandContext() reference.
  * WorkerMailbox, id = workerId. Commands: post(PeerMessage), reply(messageId,
    text), getMailbox. Events: MessagePosted, MessageReplied. State is a
    List<PeerMessage>.
  * IntakeQueue, single instance "default". Command submitGoal(GoalBrief)
    emitting GoalSubmitted{goalId, title, description, requestedBy, submittedAt}.

- 1 KeyValueEntity SystemControl, single instance "default", holding
  {halted: boolean, haltedReason: Optional<String>, haltedBy: Optional<String>,
  haltedAt: Optional<Instant>}. Commands: halt(reason, by), resume, getControl.

- 1 View TaskBoardView with row type TaskRow (mirrors Task minus the heavy
  OutputSection contents — keep artifactSummary and sectionCount only). Table
  updater consumes TaskEntity events. ONE query getAllTasks SELECT * AS tasks FROM
  task_board. No WHERE status filter (Akka cannot auto-index enum columns —
  Lesson 2); callers filter client-side.

- 1 Consumer GoalRequestConsumer subscribed to IntakeQueue events; on
  GoalSubmitted calls GoalEntity.createGoal then starts a PlanningWorkflow with
  the goalId as the workflow id.

- 2 TimedActions:
  * GoalSimulator — every 60s, reads the next line from
    src/main/resources/sample-events/goal-briefs.jsonl and calls
    IntakeQueue.submitGoal. Stops cycling when the file is exhausted (wraps).
  * StuckTaskMonitor — every 30s, queries TaskBoardView.getAllTasks, finds
    tasks in CLAIMED or IN_PROGRESS whose claimedAt is older than 2 minutes,
    and calls TaskEntity.release (-> OPEN, clears claimedBy).

- 2 HttpEndpoints:
  * SharedTaskEndpoint at /api with POST /goals, GET /goals/{id},
    GET /tasks (client-side filter by goalId/status), GET /tasks/sse,
    GET /mailbox/{workerId}, POST /mailbox/{workerId}/messages/{id}/reply,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 service-setup Bootstrap: schedule GoalSimulator and StuckTaskMonitor;
  start one WorkerWorkflow instance per worker id (worker-1, worker-2, worker-3).

Companion files:
- TeamTasks.java declaring two Task<R> constants: PLAN (resultConformsTo
  TaskPlan) and WORK (resultConformsTo ResultArtifact).
- Domain records GoalBrief, TaskSpec, TaskPlan, OutputSection, PeerRequest,
  ResultArtifact, QualityReport, PeerMessage in domain/.
- QualityChecker.java (deterministic, in application/) — a pure function over a
  ResultArtifact returning a QualityReport: passes when every OutputSection has a
  non-blank heading and content, the summary is non-empty, and no section content
  contains a literal "TODO" or "PLACEHOLDER"; otherwise fails with one failure
  string per offending section. This is the quality check (not an LLM call).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9881 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also a worker-roster setting
  shared-task-team.workers = ["worker-1","worker-2","worker-3"] read by Bootstrap.
- src/main/resources/sample-events/goal-briefs.jsonl with 6 canned goal briefs
  (each a small, self-contained coordination goal — e.g., research and summarise a
  topic, draft a structured report, produce a comparative analysis, generate an FAQ
  document, outline a workshop agenda, write a concise briefing).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (G1
  before-agent-response guardrail, E1 on-decision eval, HT1 halt
  operator-regulator-stop) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/goal-planner.md and prompts/worker-agent.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Shared Task Team",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual" prefix
  on tab names.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs matching the formal exemplar:
  Overview, Architecture (4 mermaid diagrams + click-to-expand component table
  with syntax-highlighted Java snippets), Risk Survey (7 sections from
  governance.html with answers from risk-survey.yaml; unanswered .qb opacity
  0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table
  with click-to-expand rows and a colored mechanism pill in the ID column), App
  UI (goal form + a board with one column per TaskStatus, a peer-mailbox panel
  with reply boxes, and a Halt/Resume control). Browser title exactly:
  <title>Akka Sample: Shared Task Team</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via
  the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider block
        below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the
        session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the value
  lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing the
  ModelProvider interface with a per-agent dispatch on the agent class name (or
  the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (goal-planner.json,
  worker-agent.json), picks one entry pseudo-randomly per call, and deserialises
  it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    goal-planner.json — 4-6 TaskPlan entries. Each has a planSummary (one
      sentence) and 3-6 TaskSpec items with believable tasks for a small
      coordination goal (e.g., "Research background on the topic",
      "Draft the introduction section", "Write the comparative analysis",
      "Compile the bibliography", "Produce the executive summary"), with
      dependsOn referencing earlier task titles to exercise the dependency gate.
    worker-agent.json — 4-6 ResultArtifact entries. Most have 1-3 OutputSection
      items with a plausible heading (e.g., "Background") and short but
      non-empty, placeholder-free content plus a one-sentence summary and an
      empty peerRequest. Include 1-2 entries whose peerRequest is present
      (toWorker "worker-2", a concrete question) to exercise the
      peer-coordination journey, and include 1 entry whose content contains a
      literal "PLACEHOLDER" so the quality check fails deterministically for
      J-quality-gate coverage.
- A MockModelProvider.seedFor(taskId) helper makes the selection deterministic
  per task id so the same task in dev produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion TeamTasks Task<R> constants (Lesson 7).
- View has no WHERE filter on the enum status column; filter client-side
  (Lesson 2).
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9881 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden" zombie
  panels in the DOM — delete removed tabs, do not display:none them.
- UI is a single self-contained static-resources/index.html — no ui/ folder, no
  package.json, no npm build (Lesson 17).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, use, use, marketing
  tone, competitor brand names (Lessons 21, 22, 23).
- Lessons referenced in this SPEC: 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24,
  25, 26.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars plus the mock-LLM option from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
