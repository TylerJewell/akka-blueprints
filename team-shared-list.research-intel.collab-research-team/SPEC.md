# SPEC — collab-research-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Collaborative Research Assistant.
**One-line pitch:** Submit a research question; a coordinator agent fans it out to specialized researchers who each claim a sub-topic and gather findings, then a synthesis agent merges all findings into a single cited report that an eval agent verifies for citation grounding.

## 2. What this blueprint demonstrates

The **team-shared-list** coordination pattern wired with Akka's first-party primitives: a planning workflow runs a coordinator agent that writes a list of sub-topic tasks onto a shared board, and a fixed roster of researcher agents each runs its own durable loop that pulls open tasks off that board, claims them atomically, produces a findings report, and marks them done. No central dispatcher hands work out — the researchers self-organise around the shared list, and the board entity's single-writer guarantee prevents two researchers claiming the same sub-topic. The blueprint also demonstrates two governance mechanisms a research pipeline requires: a **citation-grounding eval** that fires on each decision the synthesis agent makes (verifying every conclusion cites at least one source), and an **after-llm-response guardrail** that blocks a final report whose hallucination score exceeds a threshold.

## 3. User-facing flows

The user opens the App UI tab and submits a research question via the form (a question title plus any context notes).

1. The system logs the submission on `IntakeQueue` and a `Consumer` starts a `PlanningWorkflow` for the question.
2. The `ResearchCoordinator` agent decomposes the question into a `ResearchPlan` — a list of sub-topics, each with a focus statement and a list of keywords. The workflow writes one `ResearchTaskEntity` per sub-topic (status `OPEN`) and moves the question to `READY`.
3. The researcher roster (`researcher-1`, `researcher-2`, `researcher-3`) is already running — each researcher is a `ResearcherWorkflow` that polls the shared `TaskBoardView` for an `OPEN` task.
4. A researcher claims an eligible task. The claim is an atomic command on that task's `ResearchTaskEntity`; if two researchers race for the same task, one wins and the other goes back to polling.
5. The winning researcher runs the `ResearcherAgent`, which gathers sources and returns a `FindingsReport` (a list of `SourceRecord` items plus a summary). Every tool call passes through the after-llm-response guardrail; a report whose citation count is below the minimum is refused and the task is recorded `BLOCKED`.
6. When every task on a question is `DONE`, a `SynthesisWorkflow` starts. It runs the `SynthesisAgent`, which merges all `FindingsReport` items into a single `ResearchReport`. An eval event fires for each conclusion in the merged report, verifying citation grounding.
7. If any conclusion fails citation grounding, the report is flagged `NEEDS_REVIEW`; an operator can approve or reject it. An approved report moves the question to `COMPLETED`.
8. If a researcher needs clarification from another researcher, it raises a `CoordinationRequest`. The task goes `BLOCKED`, a `CoordinationMessage` lands in the target researcher's `ResearcherMailbox`, and the researcher who can answer posts a reply that unblocks the waiting task.

A `QuestionSimulator` (TimedAction) drips a sample research question every 90 seconds so the App UI is not empty when first loaded. A `StuckTaskMonitor` (TimedAction) releases any task that has been claimed but idle for more than three minutes, returning it to `OPEN`.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchCoordinator` | `AutonomousAgent` | Decomposes a research question into a list of focused sub-topics with keywords. | `PlanningWorkflow` | returns typed `ResearchPlan` to workflow |
| `ResearcherAgent` | `AutonomousAgent` | Claims and researches one sub-topic; returns a `FindingsReport`; may raise a `CoordinationRequest`. Run as several instances (`researcher-1`, `researcher-2`, `researcher-3`). | `ResearcherWorkflow` | returns typed result to workflow |
| `SynthesisAgent` | `AutonomousAgent` | Merges all `FindingsReport` items for a question into a single `ResearchReport` with citations. | `SynthesisWorkflow` | returns typed `ResearchReport` to workflow |
| `PlanningWorkflow` | `Workflow` | Runs `ResearchCoordinator`, creates one `ResearchTaskEntity` per sub-topic, moves the question to `READY`. | `QuestionRequestConsumer` | `ResearchCoordinator`, `ResearchTaskEntity`, `QuestionEntity` |
| `ResearcherWorkflow` | `Workflow` | Per-researcher durable loop: poll → claim → research → grounding check → done/blocked/coordination. | `Bootstrap` (one instance per researcher id) | `ResearchTaskEntity`, `ResearcherAgent`, `ResearcherMailbox`, `OperatorControl` |
| `SynthesisWorkflow` | `Workflow` | Starts when all sub-topic tasks for a question are `DONE`; runs `SynthesisAgent`; fires eval events per conclusion; moves question to `COMPLETED` or `NEEDS_REVIEW`. | `QuestionCompletionMonitor` | `SynthesisAgent`, `QuestionEntity` |
| `QuestionEntity` | `EventSourcedEntity` | Question lifecycle: brief, task ids, synthesis report, status. | `PlanningWorkflow`, `SynthesisWorkflow`, `QuestionCompletionMonitor` | endpoint reads |
| `ResearchTaskEntity` | `EventSourcedEntity` | One per sub-topic task. Atomic `claim`; full task lifecycle. | `PlanningWorkflow`, `ResearcherWorkflow`, `StuckTaskMonitor` | `TaskBoardView` |
| `ResearcherMailbox` | `EventSourcedEntity` | One per researcher. Inbound coordination messages and replies. | `ResearcherWorkflow` | `ResearchEndpoint` |
| `IntakeQueue` | `EventSourcedEntity` | Single instance; logs each submitted question for replay/audit. | `ResearchEndpoint`, `QuestionSimulator` | `QuestionRequestConsumer` |
| `OperatorControl` | `KeyValueEntity` | Operator halt flag for the whole research team. | `ResearchEndpoint` | `ResearcherWorkflow`, `ResearcherAgent` guardrail |
| `TaskBoardView` | `View` | The shared task list read model the researchers poll and the UI streams. | `ResearchTaskEntity` events | `ResearcherWorkflow`, `ResearchEndpoint` |
| `QuestionRequestConsumer` | `Consumer` | Listens to `IntakeQueue` events; starts a `PlanningWorkflow` per submission. | `IntakeQueue` events | `PlanningWorkflow` |
| `QuestionCompletionMonitor` | `TimedAction` | Every 30 s, checks whether all tasks for a question are `DONE` and starts `SynthesisWorkflow` if so. | scheduler | `QuestionEntity`, `SynthesisWorkflow` |
| `QuestionSimulator` | `TimedAction` | Drips a sample research question every 90 s. | scheduler | `IntakeQueue` |
| `StuckTaskMonitor` | `TimedAction` | Every 30 s, releases tasks claimed but idle > 3 min back to `OPEN`. | scheduler | `ResearchTaskEntity` |
| `ResearchEndpoint` | `HttpEndpoint` | `/api/*` — submit, list tasks, get question, coordination reply, halt/resume, SSE. | — | `IntakeQueue`, `TaskBoardView`, `QuestionEntity`, `ResearcherMailbox`, `OperatorControl` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the three TimedActions; starts one `ResearcherWorkflow` per researcher id. | — | `ResearcherWorkflow`, scheduler |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record ResearchQuestion(String questionId, String title, String context, String submittedBy) {}

record SubTopicSpec(String title, String focusStatement, List<String> keywords) {}
record ResearchPlan(String planSummary, List<SubTopicSpec> subTopics) {}

record SourceRecord(String url, String title, String excerpt) {}
record CoordinationRequest(String toResearcher, String question) {}
record FindingsReport(String taskId, List<SourceRecord> sources, String summary,
                      Optional<CoordinationRequest> coordinationRequest) {}

record ReportConclusion(String statement, List<String> citedUrls) {}
record ResearchReport(String questionId, List<ReportConclusion> conclusions,
                      String executiveSummary, Instant synthesizedAt) {}

record CitationEvalResult(String statement, boolean grounded, List<String> missingCitations) {}
record CoordinationMessage(String messageId, String fromResearcher, String toResearcher,
                           String taskId, String question, Instant sentAt,
                           Optional<String> reply, Optional<Instant> repliedAt) {}
```

### Entity state — `ResearchTask` (state of `ResearchTaskEntity`, basis of the view row)

```java
record ResearchTask(
    String taskId,
    String questionId,
    String title,
    String focusStatement,
    List<String> keywords,
    ResearchTaskStatus status,
    Optional<String>  claimedBy,
    Optional<Instant> claimedAt,
    Optional<String>  findingsSummary,
    Optional<Integer> sourceCount,
    Optional<String>  blockedReason,
    Optional<Instant> completedAt,
    Instant createdAt
) {}

enum ResearchTaskStatus { OPEN, CLAIMED, IN_PROGRESS, FINDINGS_READY, DONE, BLOCKED }
```

### Entity state — `Question` (state of `QuestionEntity`)

```java
record Question(
    String questionId,
    String title,
    String context,
    String submittedBy,
    QuestionStatus status,
    List<String> taskIds,
    Optional<String> planSummary,
    Optional<ResearchReport> report,
    Optional<String> reviewNote,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum QuestionStatus { PLANNING, READY, IN_PROGRESS, SYNTHESIZING, NEEDS_REVIEW, COMPLETED }
```

### Events

`ResearchTaskEntity`: `TaskCreated`, `TaskClaimed`, `TaskStarted`, `FindingsSubmitted`, `TaskCompleted`, `TaskBlocked`, `TaskReleased`.
`QuestionEntity`: `QuestionCreated`, `QuestionPlanned`, `QuestionStarted`, `SynthesisStarted`, `ReportReady`, `ReportApproved`, `ReportRejected`.
`ResearcherMailbox`: `MessagePosted`, `MessageReplied`.
`IntakeQueue`: `QuestionSubmitted`.
`OperatorControl` is a `KeyValueEntity` — state transitions are command-driven, no events.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/questions` — body `{ title, context, submittedBy? }` → `{ questionId }`. Logs the submission and starts planning.
- `GET /api/questions/{id}` — one question with its task ids, status, and report if available.
- `GET /api/tasks` — list all tasks. Optional `?questionId=...` and `?status=...`, filtered client-side.
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `GET /api/mailbox/{researcherId}` — coordination messages for a researcher.
- `POST /api/mailbox/{researcherId}/messages/{messageId}/reply` — body `{ reply }`. Posts a reply that unblocks the waiting task.
- `POST /api/questions/{id}/approve` — body `{ reviewedBy, note? }`. Approves a `NEEDS_REVIEW` report.
- `POST /api/questions/{id}/reject` — body `{ reviewedBy, reason }`. Rejects a `NEEDS_REVIEW` report.
- `POST /api/control/halt` — body `{ reason, by }`. Sets the operator halt flag.
- `POST /api/control/resume` — clears the halt flag.
- `GET /api/control` — current halt state.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Collaborative Research Assistant</title>`.

- **Overview** — eyebrow "Overview" + headline "Collaborative Research Assistant"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, task state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a question submission form, a live board grouped by status column (Open / Claimed / In progress / Findings Ready / Done / Blocked), per-task cards showing the claiming researcher and findings summary, a researcher mailbox panel with reply boxes, a final report viewer when the question is completed, and a Halt / Resume control.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — citation-grounding eval** (`eval-event` flavor `on-decision-eval`, on each `ReportConclusion` the `SynthesisAgent` produces): every conclusion is evaluated against the cited source URLs; a conclusion with no grounding citations is flagged and the question moves to `NEEDS_REVIEW` rather than `COMPLETED`. Non-blocking at the inference step; the result gates the question status transition.
- **G1 — after-llm-response guardrail** (`guardrail` flavor `after-llm-response`, on `ResearcherAgent`): the LLM's raw `FindingsReport` is checked for minimum source count (at least two distinct cited URLs per sub-topic); if the report does not meet the threshold, the agent's response is blocked, the task is recorded `BLOCKED`, and the researcher returns to polling.

## 9. Agent prompts

- `ResearchCoordinator` → `prompts/research-coordinator.md`. Decomposes a research question into focused sub-topics.
- `ResearcherAgent` → `prompts/researcher.md`. Claims a sub-topic, gathers sources, and returns a findings report.
- `SynthesisAgent` → `prompts/synthesis-agent.md`. Merges findings reports from all researchers into one cited synthesis report.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a question; the coordinator writes sub-topic tasks to the board; researchers claim and complete them; synthesis produces a final report; the question reaches `COMPLETED`. The board updates live via SSE.
2. **J2** — Two researchers poll at the same instant; each task ends up claimed by exactly one researcher (no double-claim).
3. **J3** — A researcher raises a coordination request; the task goes `BLOCKED`, a message lands in the peer's mailbox, and a reply unblocks it.
4. **J4** — The synthesis agent produces a conclusion without supporting citations; the citation-grounding eval fires and the question moves to `NEEDS_REVIEW`; an operator approves it to `COMPLETED`.
5. **J5** — A researcher's `FindingsReport` has fewer than two sources; the after-llm-response guardrail blocks it and the task is recorded `BLOCKED`.
6. **J6** — The operator halts the team; claiming pauses and tool calls are refused; resume continues the research.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named collab-research-team demonstrating the
team-shared-list × research-intel cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact team-shared-list-research-intel-collab-research-team.
Java package io.akka.samples.researchassistant. Akka 3.6.0. HTTP port 9755.

Components to wire (exactly):
- 3 AutonomousAgents:
  * ResearchCoordinator — definition() with capability(ResearchTasks.of(PLAN)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/research-coordinator.md.
    Returns ResearchPlan{planSummary, subTopics: List<SubTopicSpec{title, focusStatement,
    keywords: List<String>}>}.
  * ResearcherAgent — definition() with capability(ResearchTasks.of(RESEARCH)
    .maxIterationsPerTask(3)). System prompt from prompts/researcher.md. Returns
    FindingsReport{taskId, sources: List<SourceRecord{url, title, excerpt}>, summary,
    coordinationRequest: Optional<CoordinationRequest{toResearcher, question}>}.
    Run as several runtime instances addressed by instanceId (researcher-1, researcher-2,
    researcher-3) — ONE agent class, several instance ids; never generate three agent classes.
    Register an after-llm-response guardrail on this agent (control G1).
  * SynthesisAgent — definition() with capability(ResearchTasks.of(SYNTHESIZE)
    .maxIterationsPerTask(2)). System prompt from prompts/synthesis-agent.md. Returns
    ResearchReport{questionId, conclusions: List<ReportConclusion{statement, citedUrls}>,
    executiveSummary, synthesizedAt}. Fire an on-decision-eval event per conclusion
    in SynthesisWorkflow (control E1).

- 1 Workflow PlanningWorkflow with steps planStep -> createTasksStep -> readyStep.
  planStep calls forAutonomousAgent(ResearchCoordinator.class, questionId)
  .runSingleTask(PLAN.instructions(question)) then forTask(taskId).result(PLAN).
  createTasksStep writes one ResearchTaskEntity per SubTopicSpec with a deterministic
  taskId = questionId + "-t" + index and calls QuestionEntity.recordPlan(taskIds, planSummary).
  readyStep ends the workflow.
  Override settings() with stepTimeout(planStep, 90s).

- 1 Workflow ResearcherWorkflow, one durable instance per researcher id. State carries
  researcherId and the currently-claimed taskId (Optional). Steps:
  pollStep -> claimStep -> researchStep -> finishStep, with a self-scheduled resume timer
  for idling. pollStep: read OperatorControl; if halted, schedule a 5s resume timer and
  pause. Otherwise query TaskBoardView.getAllTasks, pick the oldest OPEN task; if none,
  schedule a 5s resume timer and pause; else go to claimStep. claimStep: call
  ResearchTaskEntity.claim(researcherId); if the entity reports the task is no longer
  OPEN (lost the race), go back to pollStep; else ResearchTaskEntity.start and go to
  researchStep. researchStep (stepTimeout 90s): call
  forAutonomousAgent(ResearcherAgent.class, researcherId).runSingleTask(
  RESEARCH.instructions(task)). If a guardrail refusal comes back (G1 — too few sources),
  call ResearchTaskEntity.block(reason) and go to pollStep. On a CoordinationRequest in
  the result, call ResearcherMailbox(toResearcher).post(message) and
  ResearchTaskEntity.block("waiting on: " + toResearcher), then go to pollStep. Otherwise
  call ResearchTaskEntity.submitFindings(report) then ResearchTaskEntity.complete and go
  to finishStep. finishStep: clear the claimed task and go to pollStep.

- 1 Workflow SynthesisWorkflow, started by QuestionCompletionMonitor per questionId.
  Steps: gatherStep -> synthesizeStep -> evalStep -> recordStep. gatherStep: read all
  DONE tasks for the question from TaskBoardView; collect their FindingsReport summaries.
  synthesizeStep (stepTimeout 90s): call
  forAutonomousAgent(SynthesisAgent.class, questionId).runSingleTask(
  SYNTHESIZE.instructions(allFindings)). evalStep: for each conclusion in the report,
  fire a CitationEvalEvent and run CitationEvaluator.evaluate(conclusion); if any
  conclusion is not grounded, call QuestionEntity.flagForReview(evalResults) (-> NEEDS_REVIEW).
  Otherwise call QuestionEntity.complete(report) (-> COMPLETED). recordStep: end workflow.

- 4 EventSourcedEntities:
  * ResearchTaskEntity holding ResearchTask state. Commands: createTask, claim, start,
    submitFindings, complete, block, release, getTask. claim emits TaskClaimed ONLY when
    status==OPEN; otherwise a no-op returning current ResearchTask. ResearchTaskStatus
    enum: OPEN, CLAIMED, IN_PROGRESS, FINDINGS_READY, DONE, BLOCKED. Events: TaskCreated,
    TaskClaimed, TaskStarted, FindingsSubmitted, TaskCompleted, TaskBlocked, TaskReleased.
    emptyState() returns ResearchTask.initial("", "") with NO commandContext() reference.
  * QuestionEntity holding Question state. Commands: createQuestion, recordPlan, start,
    startSynthesis, complete, flagForReview, approveReport, rejectReport, getQuestion.
    QuestionStatus enum: PLANNING, READY, IN_PROGRESS, SYNTHESIZING, NEEDS_REVIEW, COMPLETED.
    Events: QuestionCreated, QuestionPlanned, QuestionStarted, SynthesisStarted, ReportReady,
    ReportApproved, ReportRejected. emptyState() returns Question.initial("") with NO
    commandContext() reference.
  * ResearcherMailbox, id = researcherId. Commands: post(CoordinationMessage),
    reply(messageId, text), getMailbox. Events: MessagePosted, MessageReplied.
    State is a List<CoordinationMessage>.
  * IntakeQueue, single instance "default". Command submitQuestion(ResearchQuestion)
    emitting QuestionSubmitted{questionId, title, context, submittedBy, submittedAt}.

- 1 KeyValueEntity OperatorControl, single instance "default", holding
  {halted: boolean, haltedReason: Optional<String>, haltedBy: Optional<String>,
  haltedAt: Optional<Instant>}. Commands: halt(reason, by), resume, getControl.

- 1 View TaskBoardView with row type TaskRow (mirrors ResearchTask minus any heavy
  content — keep findingsSummary and sourceCount only). Table updater consumes
  ResearchTaskEntity events. ONE query getAllTasks SELECT * AS tasks FROM task_board.
  No WHERE status filter (Akka cannot auto-index enum columns — Lesson 2); callers
  filter client-side.

- 1 Consumer QuestionRequestConsumer subscribed to IntakeQueue events; on
  QuestionSubmitted calls QuestionEntity.createQuestion then starts a
  PlanningWorkflow with the questionId as the workflow id.

- 3 TimedActions:
  * QuestionSimulator — every 90s, reads the next line from
    src/main/resources/sample-events/research-questions.jsonl and calls
    IntakeQueue.submitQuestion. Wraps when file is exhausted.
  * QuestionCompletionMonitor — every 30s, queries TaskBoardView.getAllTasks,
    groups by questionId, starts SynthesisWorkflow for any question whose tasks
    are all DONE and has not yet started synthesis.
  * StuckTaskMonitor — every 30s, queries TaskBoardView.getAllTasks, finds tasks
    in CLAIMED or IN_PROGRESS whose claimedAt is older than 3 minutes, and calls
    ResearchTaskEntity.release (-> OPEN, clears claimedBy).

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with POST /questions, GET /questions/{id},
    GET /tasks (client-side filter by questionId/status), GET /tasks/sse,
    GET /mailbox/{researcherId},
    POST /mailbox/{researcherId}/messages/{id}/reply,
    POST /questions/{id}/approve, POST /questions/{id}/reject,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 service-setup Bootstrap: schedule QuestionSimulator, QuestionCompletionMonitor,
  and StuckTaskMonitor; start one ResearcherWorkflow instance per researcher id
  (researcher-1, researcher-2, researcher-3).

Companion files:
- ResearchTasks.java declaring three Task<R> constants: PLAN (resultConformsTo
  ResearchPlan), RESEARCH (resultConformsTo FindingsReport), SYNTHESIZE
  (resultConformsTo ResearchReport).
- CitationEvaluator.java (deterministic, in application/) — a pure function over a
  ReportConclusion returning a CitationEvalResult: grounded when citedUrls is
  non-empty and at least one URL is present in the conclusion's statement or is a
  well-formed http/https URL; otherwise not grounded with a list of the unfounded
  claims. This is the citation gate (E1); it is NOT an LLM call.
- Domain records ResearchQuestion, SubTopicSpec, ResearchPlan, SourceRecord,
  CoordinationRequest, FindingsReport, ReportConclusion, ResearchReport,
  CitationEvalResult, CoordinationMessage in domain/.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9755
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also a
  researcher-roster setting research.researchers = ["researcher-1","researcher-2",
  "researcher-3"] read by Bootstrap.
- src/main/resources/sample-events/research-questions.jsonl with 6 canned research
  questions (e.g., "What are the leading causes of urban heat islands?",
  "How do central banks use forward guidance?", "What factors drive antibiotic
  resistance?", "What are the trade-offs in renewable energy storage?",
  "How do recommendation algorithms affect media consumption?",
  "What causes long-term trends in income inequality?").
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 2 controls (E1 citation-grounding
  eval, G1 after-llm-response guardrail) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/research-coordinator.md, prompts/researcher.md, and
  prompts/synthesis-agent.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Collaborative Research
  Assistant", one-line pitch, prerequisites (host software: none),
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs matching the formal exemplar:
  Overview, Architecture (4 mermaid diagrams + click-to-expand component table
  with syntax-highlighted Java snippets), Risk Survey (7 sections from
  governance.html with answers from risk-survey.yaml; unanswered .qb opacity 0.45),
  Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows and a colored mechanism pill in the ID column), App UI
  (question form + a board with one column per ResearchTaskStatus, a researcher
  mailbox panel with reply boxes, a final report viewer, and a Halt/Resume control).
  Browser title exactly: <title>Akka Sample: Collaborative Research Assistant</title>.
  No subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (research-coordinator.json,
  researcher.json, synthesis-agent.json), picks one entry pseudo-randomly per call,
  and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    research-coordinator.json — 4-6 ResearchPlan entries. Each has a planSummary
      (one sentence) and 3-4 SubTopicSpec items with believable sub-topics for the
      research question (e.g., "Historical context of urban heat islands",
      "Mitigation strategies", "Data sources and measurement"), with keywords lists.
    researcher.json — 4-6 FindingsReport entries. Most have 2-4 SourceRecord items
      with plausible URLs and excerpts plus a one-sentence summary and an empty
      coordinationRequest. Include 1-2 entries whose coordinationRequest is present
      (toResearcher "researcher-2", a concrete question) to exercise the coordination
      journey, and include 1 entry with only one source (triggering G1 guardrail
      refusal for the minimum-source check).
    synthesis-agent.json — 3-5 ResearchReport entries. Most have 2-4 conclusions
      each with a non-empty citedUrls list. Include 1 entry whose first conclusion
      has an empty citedUrls list to trigger the E1 citation-grounding eval failure
      and move the question to NEEDS_REVIEW.
- A MockModelProvider.seedFor(taskId) helper makes the selection deterministic per
  task id so the same task in dev produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion ResearchTasks Task<R> constants (Lesson 7).
- View has no WHERE filter on the enum status column; filter client-side (Lesson 2).
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9755 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS overrides
  AND theme variables from Lesson 24 (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc). Without these,
  state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab / data-panel
  attribute, NEVER by NodeList index (Lesson 26). No "hidden" zombie panels in
  the DOM — delete removed tabs, do not display:none them.
- UI is a single self-contained static-resources/index.html — no ui/ folder, no
  package.json, no npm build (Lesson 17).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export
  block. Per Lesson 25, /akka:specify handles the key during generation.
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
