# SPEC — dev-team-task-board

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Dev Team Task Board.
**One-line pitch:** Submit a software project; a team lead agent splits it into tasks on a shared board, and several developer agents claim tasks on their own, write code, run a test gate, and message each other when they need to coordinate.

## 2. What this blueprint demonstrates

The **team-shared-list** coordination pattern wired with Akka's first-party primitives: a planning workflow runs a team-lead agent that writes a list of tasks onto a shared board, and a fixed roster of developer agents each runs its own durable loop that pulls open tasks off that board, claims them atomically, produces a code artifact, and marks them done. No central dispatcher hands work out — the developers self-organise around the shared list, and the board entity's single-writer guarantee is what stops two developers claiming the same task. The blueprint also demonstrates three governance mechanisms a self-organising dev team needs: a **before-tool-call guardrail** that refuses destructive workspace operations, a **test gate** that a task must pass before it counts as done, and an **operator halt** that freezes the whole team.

## 3. User-facing flows

The user opens the App UI tab and submits a project via the form (a title plus a free-text description).

1. The system logs the submission on `IntakeQueue` and a `Consumer` starts a `PlanningWorkflow` for the project.
2. The `TeamLead` agent decomposes the brief into a `TaskPlan` — a list of tasks, each with a title, a description, and the titles of tasks it depends on. The workflow writes one `TaskEntity` per task (status `OPEN`) and moves the project to `READY`.
3. The developer roster (`dev-1`, `dev-2`, `dev-3`) is already running — each developer is a `DeveloperWorkflow` that polls the shared `TaskBoardView` for an `OPEN` task whose dependencies are all `DONE`.
4. A developer claims an eligible task. The claim is an atomic command on that task's `TaskEntity`; if two developers race for the same task, one wins and the other goes back to polling.
5. The winning developer runs the `DeveloperAgent`, which writes a `CodeArtifact` (a set of files plus a summary). Every workspace tool call passes through a before-tool-call guardrail; a destructive operation is refused and the task is recorded `BLOCKED`.
6. The artifact runs through a test gate. On pass, the task becomes `DONE`. On fail, the developer retries up to a fixed limit, then the task becomes `BLOCKED`.
7. If the agent reports it needs another developer's output, it raises a `PeerRequest`. The task goes `BLOCKED`, a `PeerMessage` lands in the target developer's `PeerMailbox`, and the developer who can answer posts a reply that unblocks the waiting task.
8. When every task on a project is `DONE`, the project moves to `COMPLETED`.

A `ProjectSimulator` (TimedAction) drips a sample project brief every 60 seconds so the App UI is not empty when first loaded. A `StuckTaskMonitor` (TimedAction) releases any task that has been claimed but idle for more than two minutes, returning it to `OPEN` so another developer can pick it up.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TeamLead` | `AutonomousAgent` | Decomposes a project brief into a list of tasks with dependencies. | `PlanningWorkflow` | returns typed `TaskPlan` to workflow |
| `DeveloperAgent` | `AutonomousAgent` | Claims and codes one task; returns a `CodeArtifact`; may raise a `PeerRequest`. Run as several instances (`dev-1`, `dev-2`, `dev-3`). | `DeveloperWorkflow` | returns typed result to workflow |
| `PlanningWorkflow` | `Workflow` | Runs `TeamLead`, creates one `TaskEntity` per task, moves the project to `READY`. | `ProjectRequestConsumer` | `TeamLead`, `TaskEntity`, `ProjectEntity` |
| `DeveloperWorkflow` | `Workflow` | Per-developer durable loop: poll → claim → code → test gate → done/blocked/peer. | `Bootstrap` (one instance per developer id) | `TaskEntity`, `DeveloperAgent`, `PeerMailbox`, `SystemControl` |
| `ProjectEntity` | `EventSourcedEntity` | Project lifecycle: brief, task ids, status. | `PlanningWorkflow`, `ProjectCompletionMonitor` | `TaskBoardView` is task-side; project read via endpoint |
| `TaskEntity` | `EventSourcedEntity` | One per task. Atomic `claim`; full task lifecycle. | `PlanningWorkflow`, `DeveloperWorkflow`, `StuckTaskMonitor` | `TaskBoardView` |
| `PeerMailbox` | `EventSourcedEntity` | One per developer. Inbound peer messages and replies. | `DeveloperWorkflow` | `DevTeamEndpoint` |
| `IntakeQueue` | `EventSourcedEntity` | Single instance; logs each submitted project for replay/audit. | `DevTeamEndpoint`, `ProjectSimulator` | `ProjectRequestConsumer` |
| `SystemControl` | `KeyValueEntity` | Operator halt flag for the whole team. | `DevTeamEndpoint` | `DeveloperWorkflow`, `DeveloperAgent` guardrail |
| `TaskBoardView` | `View` | The shared task list read model the developers poll and the UI streams. | `TaskEntity` events | `DeveloperWorkflow`, `DevTeamEndpoint` |
| `ProjectRequestConsumer` | `Consumer` | Listens to `IntakeQueue` events; starts a `PlanningWorkflow` per submission. | `IntakeQueue` events | `PlanningWorkflow` |
| `ProjectSimulator` | `TimedAction` | Drips a sample project brief every 60 s. | scheduler | `IntakeQueue` |
| `StuckTaskMonitor` | `TimedAction` | Every 30 s, releases tasks claimed but idle > 2 min back to `OPEN`. | scheduler | `TaskEntity` |
| `DevTeamEndpoint` | `HttpEndpoint` | `/api/*` — submit, list tasks, get project, peer reply, halt/resume, SSE. | — | `IntakeQueue`, `TaskBoardView`, `ProjectEntity`, `PeerMailbox`, `SystemControl` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions; starts one `DeveloperWorkflow` per developer id. | — | `DeveloperWorkflow`, scheduler |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record ProjectBrief(String projectId, String title, String description, String requestedBy) {}

record TaskSpec(String title, String description, List<String> dependsOn) {}
record TaskPlan(String planSummary, List<TaskSpec> tasks) {}

record CodeFile(String path, String content) {}
record PeerRequest(String toDeveloper, String question) {}
record CodeArtifact(String taskId, List<CodeFile> files, String summary, Optional<PeerRequest> peerRequest) {}

record TestReport(boolean passed, List<String> failures, Instant ranAt) {}
record PeerMessage(String messageId, String fromDeveloper, String toDeveloper, String taskId,
                   String question, Instant sentAt, Optional<String> reply, Optional<Instant> repliedAt) {}
```

### Entity state — `Task` (state of `TaskEntity`, basis of the view row)

```java
record Task(
    String taskId,
    String projectId,
    String title,
    String description,
    List<String> dependsOn,
    TaskStatus status,
    Optional<String>  claimedBy,
    Optional<Instant> claimedAt,
    Optional<String>  artifactSummary,
    Optional<Integer> fileCount,
    Optional<TestReport> testReport,
    Optional<String>  blockedReason,
    Optional<Instant> completedAt,
    Instant createdAt
) {}

enum TaskStatus { OPEN, CLAIMED, IN_PROGRESS, IN_REVIEW, DONE, BLOCKED }
```

### Entity state — `Project` (state of `ProjectEntity`)

```java
record Project(
    String projectId,
    String title,
    String description,
    String requestedBy,
    ProjectStatus status,
    List<String> taskIds,
    Optional<String> planSummary,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum ProjectStatus { PLANNING, READY, IN_PROGRESS, COMPLETED }
```

### Events

`TaskEntity`: `TaskCreated`, `TaskClaimed`, `TaskStarted`, `ArtifactSubmitted`, `TestPassed`, `TestFailed`, `TaskBlocked`, `TaskReleased`, `TaskCompleted`.
`ProjectEntity`: `ProjectCreated`, `ProjectPlanned`, `ProjectStarted`, `ProjectCompleted`.
`PeerMailbox`: `MessagePosted`, `MessageReplied`.
`IntakeQueue`: `ProjectSubmitted`.
`SystemControl` is a `KeyValueEntity` — state transitions are command-driven, no events.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/projects` — body `{ title, description, requestedBy? }` → `{ projectId }`. Logs the submission and starts planning.
- `GET /api/projects/{id}` — one project with its task ids and status.
- `GET /api/tasks` — list all tasks. Optional `?projectId=...` and `?status=...`, filtered client-side.
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `GET /api/mailbox/{developerId}` — peer messages for a developer.
- `POST /api/mailbox/{developerId}/messages/{messageId}/reply` — body `{ reply }`. Posts a reply that unblocks the waiting task.
- `POST /api/control/halt` — body `{ reason, by }`. Sets the operator halt flag.
- `POST /api/control/resume` — clears the halt flag.
- `GET /api/control` — current halt state.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Dev Team Task Board</title>`.

- **Overview** — eyebrow "Overview" + headline "Dev Team Task Board"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, task state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a project submission form, a live board grouped by status column (Open / Claimed / In progress / In review / Done / Blocked), per-task cards showing the claiming developer and artifact summary, a peer-mailbox panel with reply boxes, and a Halt / Resume control.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`tool-permission` flavor, on `DeveloperAgent`): every workspace tool call (file write, shell command, external service call) passes through a guardrail that refuses destructive operations — deleting files outside the task workspace, recursive deletes, force-push, dropping a datastore, or any call while the team is halted. A refused call records the task `BLOCKED` with the guardrail reason. Blocking.
- **A1 — test gate** (`ci-gate`, `test-gate` flavor): a task's code artifact must pass the simulated test runner before the task counts as `DONE`; the same gate is enforced at build time by the Maven `test` phase, which must be green before any deploy. Build-gate.
- **HT1 — operator halt** (`halt`, `operator-regulator-stop` flavor): an operator can freeze the whole team through `SystemControl`; `DeveloperWorkflow` stops claiming and the before-tool-call guardrail refuses every tool call until resume. System-level.

## 9. Agent prompts

- `TeamLead` → `prompts/team-lead.md`. Decomposes a project brief into a dependency-ordered task list.
- `DeveloperAgent` → `prompts/developer.md`. Claims a task, writes the code artifact, raises a peer request when it needs another developer's output.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a project; the team lead writes tasks to the board; developers claim and complete them; the project reaches `COMPLETED`. The board updates live via SSE.
2. **J2** — Two developers poll at the same instant; each task ends up claimed by exactly one developer (no double-claim).
3. **J3** — A developer raises a peer request; the task goes `BLOCKED`, a message lands in the peer's mailbox, and a reply unblocks it.
4. **J4** — A developer attempts a destructive tool call; the before-tool-call guardrail refuses it and the task is recorded `BLOCKED` with the guardrail reason.
5. **J5** — The operator halts the team; claiming pauses and tool calls are refused; resume continues the work.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named dev-team-task-board demonstrating the
team-shared-list × dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact dev-team-task-board. Java package
io.akka.samples.devteamtaskboard. Akka 3.6.0. HTTP port 9714.

Components to wire (exactly):
- 2 AutonomousAgents:
  * TeamLead — definition() with capability(TaskAcceptance.of(DECOMPOSE)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/team-lead.md.
    Returns TaskPlan{planSummary, tasks: List<TaskSpec{title, description,
    dependsOn: List<String>}>}.
  * DeveloperAgent — definition() with capability(TaskAcceptance.of(IMPLEMENT)
    .maxIterationsPerTask(3)). System prompt from prompts/developer.md. Returns
    CodeArtifact{taskId, files: List<CodeFile{path, content}>, summary,
    peerRequest: Optional<PeerRequest{toDeveloper, question}>}. Run as several
    runtime instances addressed by instanceId (dev-1, dev-2, dev-3) — ONE agent
    class, several instance ids; never generate three agent classes.
    Register a before-tool-call guardrail on this agent (control G1).

- 1 Workflow PlanningWorkflow with steps decomposeStep -> createTasksStep ->
  readyStep. decomposeStep calls forAutonomousAgent(TeamLead.class, projectId)
  .runSingleTask(DECOMPOSE.instructions(brief)) then forTask(taskId)
  .result(DECOMPOSE). createTasksStep writes one TaskEntity per TaskSpec with a
  deterministic taskId = projectId + "-t" + index and calls
  ProjectEntity.recordPlan(taskIds, planSummary). readyStep ends the workflow.
  Override settings() with stepTimeout(decomposeStep, 90s).

- 1 Workflow DeveloperWorkflow, one durable instance per developer id. State
  carries developerId and the currently-claimed taskId (Optional). Steps:
  pollStep -> claimStep -> workStep -> testGateStep -> finishStep, with a
  self-scheduled resume timer for idling. pollStep: read SystemControl; if
  halted, schedule a 5s resume timer and pause. Otherwise query
  TaskBoardView.getAllTasks, pick the oldest OPEN task whose dependsOn titles
  are all DONE; if none, schedule a 5s resume timer and pause; else go to
  claimStep. claimStep: call TaskEntity.claim(developerId); if the entity
  reports the task is no longer OPEN (lost the race), go back to pollStep; else
  TaskEntity.start and go to workStep. workStep (stepTimeout 90s): call
  forAutonomousAgent(DeveloperAgent.class, developerId).runSingleTask(
  IMPLEMENT.instructions(task)). If a guardrail refusal comes back, call
  TaskEntity.block(reason) and go to pollStep. On a PeerRequest in the result,
  call PeerMailbox(toDeveloper).post(message) and TaskEntity.block("waiting on
  peer: " + toDeveloper), then go to pollStep. Otherwise call
  TaskEntity.submitArtifact(artifact) and go to testGateStep. testGateStep:
  run the deterministic TestRunner over the artifact; on pass call
  TaskEntity.passTest(report) (-> DONE) and go to finishStep; on fail up to 2
  retries call TaskEntity.failTest(report) and loop workStep; after retries
  exhausted call TaskEntity.block("tests failed") and go to pollStep.
  finishStep: clear the claimed task and go to pollStep.

- 4 EventSourcedEntities:
  * TaskEntity holding Task state. Commands: createTask, claim, start,
    submitArtifact, passTest, failTest, block, release, getTask. claim emits
    TaskClaimed ONLY when status==OPEN; otherwise it is a no-op that returns the
    current Task so the caller can detect the lost race. TaskStatus enum: OPEN,
    CLAIMED, IN_PROGRESS, IN_REVIEW, DONE, BLOCKED. Events: TaskCreated,
    TaskClaimed, TaskStarted, ArtifactSubmitted, TestPassed, TestFailed,
    TaskBlocked, TaskReleased, TaskCompleted. emptyState() returns
    Task.initial("", "") with NO commandContext() reference.
  * ProjectEntity holding Project state. Commands: createProject, recordPlan,
    start, complete, getProject. ProjectStatus enum: PLANNING, READY,
    IN_PROGRESS, COMPLETED. Events: ProjectCreated, ProjectPlanned,
    ProjectStarted, ProjectCompleted. emptyState() returns Project.initial("")
    with NO commandContext() reference.
  * PeerMailbox, id = developerId. Commands: post(PeerMessage), reply(messageId,
    text), getMailbox. Events: MessagePosted, MessageReplied. State is a
    List<PeerMessage>.
  * IntakeQueue, single instance "default". Command submitProject(ProjectBrief)
    emitting ProjectSubmitted{projectId, title, description, requestedBy,
    submittedAt}.

- 1 KeyValueEntity SystemControl, single instance "default", holding
  {halted: boolean, haltedReason: Optional<String>, haltedBy: Optional<String>,
  haltedAt: Optional<Instant>}. Commands: halt(reason, by), resume, getControl.

- 1 View TaskBoardView with row type TaskRow (mirrors Task minus the heavy
  CodeFile contents — keep artifactSummary and fileCount only). Table updater
  consumes TaskEntity events. ONE query getAllTasks SELECT * AS tasks FROM
  task_board. No WHERE status filter (Akka cannot auto-index enum columns —
  Lesson 2); callers filter client-side.

- 1 Consumer ProjectRequestConsumer subscribed to IntakeQueue events; on
  ProjectSubmitted calls ProjectEntity.createProject then starts a
  PlanningWorkflow with the projectId as the workflow id.

- 2 TimedActions:
  * ProjectSimulator — every 60s, reads the next line from
    src/main/resources/sample-events/project-briefs.jsonl and calls
    IntakeQueue.submitProject. Stops cycling when the file is exhausted (wraps).
  * StuckTaskMonitor — every 30s, queries TaskBoardView.getAllTasks, finds
    tasks in CLAIMED or IN_PROGRESS whose claimedAt is older than 2 minutes,
    and calls TaskEntity.release (-> OPEN, clears claimedBy).

- 2 HttpEndpoints:
  * DevTeamEndpoint at /api with POST /projects, GET /projects/{id},
    GET /tasks (client-side filter by projectId/status), GET /tasks/sse,
    GET /mailbox/{developerId}, POST /mailbox/{developerId}/messages/{id}/reply,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 service-setup Bootstrap: schedule ProjectSimulator and StuckTaskMonitor;
  start one DeveloperWorkflow instance per developer id (dev-1, dev-2, dev-3).

Companion files:
- TeamTasks.java declaring two Task<R> constants: DECOMPOSE (resultConformsTo
  TaskPlan) and IMPLEMENT (resultConformsTo CodeArtifact).
- Domain records ProjectBrief, TaskSpec, TaskPlan, CodeFile, PeerRequest,
  CodeArtifact, TestReport, PeerMessage in domain/.
- TestRunner.java (deterministic, in application/) — a pure function over a
  CodeArtifact returning a TestReport: passes when every CodeFile has a
  non-blank path and content, the summary is non-empty, and no file contains a
  literal "TODO" or "throw new UnsupportedOperationException"; otherwise fails
  with one failure string per offending file. This is the test gate (A1); it is
  NOT an LLM call.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9714 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also a developer-roster setting
  dev-team.developers = ["dev-1","dev-2","dev-3"] read by Bootstrap.
- src/main/resources/sample-events/project-briefs.jsonl with 6 canned project
  briefs (each a small, self-contained software project — e.g., a URL
  shortener, a CSV-to-JSON converter, a rate limiter, a markdown table
  formatter, a retry helper, a config loader).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (G1 before-tool-call
  guardrail, A1 ci-gate test-gate, HT1 halt operator-regulator-stop) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/team-lead.md and prompts/developer.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Dev Team Task Board",
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
  UI (project form + a board with one column per TaskStatus, a peer-mailbox
  panel with reply boxes, and a Halt/Resume control). Browser title exactly:
  <title>Akka Sample: Dev Team Task Board</title>. No subtitle on the Overview
  tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
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
  src/main/resources/mock-responses/<agent-name>.json (team-lead.json,
  developer.json), picks one entry pseudo-randomly per call, and deserialises it
  into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    team-lead.json — 4-6 TaskPlan entries. Each has a planSummary (one sentence)
      and 3-6 TaskSpec items with believable dev tasks for a small project
      (e.g., "Define the URL record", "Implement the encoder", "Add the lookup
      endpoint", "Write the in-memory store"), with dependsOn referencing
      earlier task titles to exercise the dependency gate.
    developer.json — 4-6 CodeArtifact entries. Most have 1-3 CodeFile items with
      a plausible path (e.g., "src/UrlShortener.java") and short but non-empty,
      TODO-free content plus a one-sentence summary and an empty peerRequest.
      Include 1-2 entries whose peerRequest is present (toDeveloper "dev-2",
      a concrete question) to exercise the peer-coordination journey, and
      include 1 entry whose content contains a literal "TODO" so the test gate
      fails deterministically for J-test-gate coverage.
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
- Explicit http-port in application.conf — 9714 (Lessons 10, 13).
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
