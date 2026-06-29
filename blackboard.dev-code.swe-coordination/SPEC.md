# SPEC — blackboard-swe-coordination

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Blackboard SWE Coordination.
**One-line pitch:** Submit a software ticket; nine specialist agents read a shared blackboard, each contributes its layer (planning, architecture, code, review, tests, security, docs, integration), and a lead-engineer signoff gates the final merge.

## 2. What this blueprint demonstrates

The **blackboard** coordination pattern wired with Akka's first-party primitives: a single `BlackboardEntity` is the shared knowledge store, and a `ControllerWorkflow` decides which specialist agent runs next by inspecting the blackboard's current stage. Each specialist reads the board, produces a typed contribution, and the controller writes it back — no specialist talks to another directly; all knowledge flows through the board. The blueprint also demonstrates two governance mechanisms: a **before-state-write guardrail** that type-checks and lint-validates code before it lands on the blackboard, and a **HITL application signoff** that pauses the pipeline for a lead-engineer merge decision.

## 3. User-facing flows

The user opens the App UI tab and submits a ticket via the form (a title plus a free-text description).

1. The system logs the submission on `TicketQueue` and a `Consumer` starts a `ControllerWorkflow` for the ticket.
2. The controller reads the blackboard stage (`INTAKE`) and invokes `PlannerAgent`, which writes a `TaskBreakdown` onto the board. Stage advances to `PLANNED`.
3. The controller invokes `ArchitectAgent`, which reads the task breakdown and writes an `ArchDecision`. Stage advances to `ARCHITECTED`.
4. The controller invokes `BackendDevAgent` and `FrontendDevAgent` in parallel (both read the arch decision; each writes its `CodeArtifact`). Stage advances to `CODED` once both are present.
5. The controller invokes `ReviewerAgent` and `SecurityAnalystAgent` in parallel (both read code artifacts). Stage advances to `REVIEWED` once both have contributed findings.
6. The controller invokes `TesterAgent` (reads artifacts + review findings) and `DocWriterAgent` (reads artifacts + decisions) in parallel. Stage advances to `VERIFIED` once both are done.
7. The controller invokes `IntegrationPlannerAgent`, which reads all prior contributions and writes an `IntegrationPlan`. Stage advances to `INTEGRATION_PLANNED`.
8. The controller starts a `SignoffWorkflow`. The workflow pauses, sending a `SignoffRequest` to the lead engineer's `SignoffEntity`. The board moves to `AWAITING_SIGNOFF`.
9. The lead engineer approves or rejects via `POST /api/signoff/{id}`. On approval the board advances to `MERGED`. On rejection it returns to `IN_REVIEW` and the controller re-invokes the reviewer.
10. When the board reaches `MERGED`, the `TicketEntity` moves to `DONE`.

A `TicketSimulator` (TimedAction) drips a sample ticket every 90 seconds so the App UI is not empty when first loaded. A `StuckBoardMonitor` (TimedAction) detects boards that have been in a non-terminal stage for more than five minutes and emits a `BoardStuck` alert visible on the UI.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Reads the ticket brief, writes a `TaskBreakdown`. | `ControllerWorkflow` | returns `TaskBreakdown` to workflow |
| `ArchitectAgent` | `AutonomousAgent` | Reads `TaskBreakdown`, writes `ArchDecision`. | `ControllerWorkflow` | returns `ArchDecision` to workflow |
| `BackendDevAgent` | `AutonomousAgent` | Reads `ArchDecision` + task list, writes backend `CodeArtifact`. | `ControllerWorkflow` | returns `CodeArtifact` to workflow |
| `FrontendDevAgent` | `AutonomousAgent` | Reads `ArchDecision` + task list, writes frontend `CodeArtifact`. | `ControllerWorkflow` | returns `CodeArtifact` to workflow |
| `ReviewerAgent` | `AutonomousAgent` | Reads all `CodeArtifact`s, writes `ReviewFindings`. | `ControllerWorkflow` | returns `ReviewFindings` to workflow |
| `SecurityAnalystAgent` | `AutonomousAgent` | Reads `CodeArtifact`s + `ArchDecision`, writes `SecurityFindings`. | `ControllerWorkflow` | returns `SecurityFindings` to workflow |
| `TesterAgent` | `AutonomousAgent` | Reads `CodeArtifact`s + `ReviewFindings`, writes `TestResults`. | `ControllerWorkflow` | returns `TestResults` to workflow |
| `DocWriterAgent` | `AutonomousAgent` | Reads `CodeArtifact`s + `ArchDecision` + `TaskBreakdown`, writes `DocArtifact`. | `ControllerWorkflow` | returns `DocArtifact` to workflow |
| `IntegrationPlannerAgent` | `AutonomousAgent` | Reads all prior contributions, writes `IntegrationPlan`. | `ControllerWorkflow` | returns `IntegrationPlan` to workflow |
| `ControllerWorkflow` | `Workflow` | Blackboard controller: inspects the current stage, invokes the right specialist(s), writes results back. One instance per ticket. | `TicketConsumer` | all nine agents, `BlackboardEntity`, `SignoffWorkflow` |
| `SignoffWorkflow` | `Workflow` | Pauses for HITL approval. Advances the board to `MERGED` on approval or `IN_REVIEW` on rejection. | `ControllerWorkflow` | `SignoffEntity`, `BlackboardEntity` |
| `BlackboardEntity` | `EventSourcedEntity` | Central knowledge store. Holds ticket, stage, and all specialist contributions as events. | `ControllerWorkflow`, `SignoffWorkflow` | `BlackboardView` |
| `TicketEntity` | `EventSourcedEntity` | Per-ticket lifecycle: `OPEN` → `IN_PROGRESS` → `DONE` / `REJECTED`. | `TicketConsumer`, `ControllerWorkflow` | `BoardEndpoint` |
| `SignoffEntity` | `EventSourcedEntity` | Per-ticket signoff record: request, decision, actor, timestamp. | `SignoffWorkflow` | `BoardEndpoint` |
| `TicketQueue` | `EventSourcedEntity` | Single-instance audit log of every submitted ticket. | `BoardEndpoint`, `TicketSimulator` | `TicketConsumer` |
| `BlackboardView` | `View` | Read-side projection of `BlackboardEntity` events. Row type `BoardRow`. | `BlackboardEntity` events | `ControllerWorkflow`, `BoardEndpoint` |
| `TicketConsumer` | `Consumer` | Listens to `TicketQueue` events; starts a `ControllerWorkflow` per `TicketSubmitted`. | `TicketQueue` events | `ControllerWorkflow`, `TicketEntity` |
| `TicketSimulator` | `TimedAction` | Drips a sample ticket every 90 s. | scheduler | `TicketQueue` |
| `StuckBoardMonitor` | `TimedAction` | Every 60 s, scans `BlackboardView` for boards idle > 5 min in a non-terminal stage and emits a `BoardStuck` alert. | scheduler | `BlackboardEntity` |
| `BoardEndpoint` | `HttpEndpoint` | `/api/*` — submit ticket, get board, list boards, signoff, SSE, alert. | — | `TicketQueue`, `BlackboardView`, `TicketEntity`, `SignoffEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions. | — | scheduler |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record TicketBrief(String ticketId, String title, String description, String submittedBy) {}

record TaskItem(String title, String description, String layer) {}
record TaskBreakdown(String summary, List<TaskItem> tasks) {}

record ArchDecision(String approach, List<String> components, List<String> constraints) {}

record CodeFile(String path, String content) {}
record CodeArtifact(String layer, List<CodeFile> files, String summary) {}

record ReviewFinding(String file, String severity, String comment) {}
record ReviewFindings(List<ReviewFinding> findings, boolean approvedForTesting) {}

record SecurityFinding(String category, String severity, String description) {}
record SecurityFindings(List<SecurityFinding> findings, boolean clearToMerge) {}

record TestCase(String name, boolean passed, String failureReason) {}
record TestResults(List<TestCase> cases, boolean allPassed) {}

record DocArtifact(String overview, List<String> apiSections, String deploymentNotes) {}

record IntegrationPlan(String approach, List<String> steps, List<String> rollbackSteps) {}

record SignoffRequest(String ticketId, String requestedBy, Instant requestedAt) {}
record SignoffDecision(String ticketId, String by, boolean approved, String notes, Instant decidedAt) {}
```

### Entity state — `Blackboard` (state of `BlackboardEntity`, basis of the view row)

```java
record Blackboard(
    String ticketId,
    String title,
    String description,
    BlackboardStage stage,
    Optional<TaskBreakdown>   taskBreakdown,
    Optional<ArchDecision>    archDecision,
    Optional<CodeArtifact>    backendArtifact,
    Optional<CodeArtifact>    frontendArtifact,
    Optional<ReviewFindings>  reviewFindings,
    Optional<SecurityFindings> securityFindings,
    Optional<TestResults>     testResults,
    Optional<DocArtifact>     docArtifact,
    Optional<IntegrationPlan> integrationPlan,
    Optional<SignoffDecision>  signoffDecision,
    Optional<String>          stuckAlert,
    Instant createdAt,
    Optional<Instant> mergedAt
) {}

enum BlackboardStage {
    INTAKE, PLANNED, ARCHITECTED, CODED, REVIEWED,
    VERIFIED, INTEGRATION_PLANNED, AWAITING_SIGNOFF,
    IN_REVIEW, MERGED
}
```

### Entity state — `Ticket` (state of `TicketEntity`)

```java
record Ticket(
    String ticketId,
    String title,
    String description,
    String submittedBy,
    TicketStatus status,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum TicketStatus { OPEN, IN_PROGRESS, DONE, REJECTED }
```

### Events

`BlackboardEntity`: `TicketReceived`, `PlanWritten`, `ArchWritten`, `BackendCodeWritten`, `FrontendCodeWritten`, `ReviewWritten`, `SecurityWritten`, `TestsWritten`, `DocsWritten`, `IntegrationPlanWritten`, `SignoffRequested`, `SignoffApproved`, `SignoffRejected`, `BoardStuckAlerted`.
`TicketEntity`: `TicketCreated`, `TicketStarted`, `TicketCompleted`, `TicketRejected`.
`SignoffEntity`: `SignoffRequestCreated`, `SignoffDecisionRecorded`.
`TicketQueue`: `TicketSubmitted`.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tickets` — body `{ title, description, submittedBy? }` → `{ ticketId }`. Logs the submission and starts the controller.
- `GET /api/tickets/{id}` — one ticket with its status.
- `GET /api/boards` — list all blackboard rows. Optional `?stage=...`, filtered client-side.
- `GET /api/boards/{ticketId}` — full blackboard for a ticket.
- `GET /api/boards/sse` — server-sent events stream of every blackboard stage change.
- `POST /api/signoff/{ticketId}` — body `{ approved: boolean, by, notes? }`. Records the lead-engineer decision.
- `GET /api/signoff/{ticketId}` — current signoff state.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Blackboard SWE Coordination</title>`.

- **Overview** — eyebrow "Overview" + headline "Blackboard SWE Coordination"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, blackboard stage machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a ticket submission form, a live board showing each ticket's blackboard as a stage pipeline (Intake → Planned → Architected → Coded → Reviewed → Verified → Integration Planned → Awaiting Signoff → Merged), per-stage contribution cards, a signoff panel with approve/reject controls, and a stuck-board alert banner.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-state-write guardrail** (`before-state-write` flavor, on `BlackboardEntity`): every code write to the blackboard passes a type-validator and a lint check before the event is persisted. A write that fails validation is refused; the `ControllerWorkflow` records a `guardedReason` and retries the agent once with the validation errors. Blocking.
- **H1 — lead-engineer application signoff** (`hitl`, `application` flavor, on `SignoffWorkflow`): after all specialists have contributed, the pipeline pauses and a `SignoffRequest` is sent to the lead engineer. The board stays in `AWAITING_SIGNOFF` until a human decision arrives via `POST /api/signoff/{ticketId}`. No merge proceeds without an explicit approval.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`.
- `ArchitectAgent` → `prompts/architect.md`.
- `BackendDevAgent` → `prompts/backend-dev.md`.
- `FrontendDevAgent` → `prompts/frontend-dev.md`.
- `ReviewerAgent` → `prompts/reviewer.md`.
- `SecurityAnalystAgent` → `prompts/security-analyst.md`.
- `TesterAgent` → `prompts/tester.md`.
- `DocWriterAgent` → `prompts/doc-writer.md`.
- `IntegrationPlannerAgent` → `prompts/integration-planner.md`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a ticket; all nine specialists contribute; the board reaches `AWAITING_SIGNOFF`; lead-engineer approves; board reaches `MERGED`. Board updates live via SSE.
2. **J2** — A specialist writes code that fails the type/lint guardrail; the write is rejected before the blackboard is updated; the workflow retries with the validation error.
3. **J3** — All specialists complete; the signoff workflow pauses; a lead engineer approves via the endpoint; the board advances to `MERGED`.
4. **J4** — A lead engineer rejects the merge; the board returns to `IN_REVIEW`; the reviewer re-runs.
5. **J5** — Two tickets are submitted concurrently; each progresses on its own blackboard with no cross-contamination.
6. **J6** — A board is idle > 5 min in a non-terminal stage; `StuckBoardMonitor` marks it `stuckAlert`; the UI shows the alert banner.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named blackboard-swe-coordination demonstrating the
blackboard × dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact blackboard-dev-code-swe-coordination.
Java package io.akka.samples.blackboardswecoordination. Akka 3.6.0.
HTTP port 9421.

Components to wire (exactly):
- 9 AutonomousAgents:
  * PlannerAgent — definition() with capability(BoardTasks.of(PLAN)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/planner.md.
    Returns TaskBreakdown{summary, tasks: List<TaskItem{title, description, layer}>}.
  * ArchitectAgent — definition() with capability(BoardTasks.of(ARCH)
    .maxIterationsPerTask(2)). System prompt from prompts/architect.md.
    Returns ArchDecision{approach, components, constraints}.
  * BackendDevAgent — definition() with capability(BoardTasks.of(BACKEND_CODE)
    .maxIterationsPerTask(3)). System prompt from prompts/backend-dev.md.
    Returns CodeArtifact{layer:"backend", files, summary}.
    Register a before-state-write guardrail on this agent (control G1).
  * FrontendDevAgent — definition() with capability(BoardTasks.of(FRONTEND_CODE)
    .maxIterationsPerTask(3)). System prompt from prompts/frontend-dev.md.
    Returns CodeArtifact{layer:"frontend", files, summary}.
    Register a before-state-write guardrail on this agent (control G1).
  * ReviewerAgent — definition() with capability(BoardTasks.of(REVIEW)
    .maxIterationsPerTask(2)). System prompt from prompts/reviewer.md.
    Returns ReviewFindings{findings, approvedForTesting}.
  * SecurityAnalystAgent — definition() with capability(BoardTasks.of(SECURITY)
    .maxIterationsPerTask(2)). System prompt from prompts/security-analyst.md.
    Returns SecurityFindings{findings, clearToMerge}.
  * TesterAgent — definition() with capability(BoardTasks.of(TEST)
    .maxIterationsPerTask(2)). System prompt from prompts/tester.md.
    Returns TestResults{cases, allPassed}.
  * DocWriterAgent — definition() with capability(BoardTasks.of(DOC)
    .maxIterationsPerTask(2)). System prompt from prompts/doc-writer.md.
    Returns DocArtifact{overview, apiSections, deploymentNotes}.
  * IntegrationPlannerAgent — definition() with capability(BoardTasks.of(INTEGRATE)
    .maxIterationsPerTask(2)). System prompt from prompts/integration-planner.md.
    Returns IntegrationPlan{approach, steps, rollbackSteps}.

- 1 Workflow ControllerWorkflow, one instance per ticketId. Steps:
  planStep -> archStep -> codeStep -> reviewStep -> verifyStep ->
  integrateStep -> signoffStep. Each step calls forAutonomousAgent(..., ticketId)
  .runSingleTask(BoardTasks.PLAN.instructions(board)) then writes the result to
  BlackboardEntity. codeStep invokes BackendDevAgent and FrontendDevAgent; the
  step waits for both results before advancing. reviewStep invokes ReviewerAgent
  and SecurityAnalystAgent in parallel. verifyStep invokes TesterAgent and
  DocWriterAgent in parallel. signoffStep starts a SignoffWorkflow(ticketId)
  and ends. Override settings() with stepTimeout(planStep, 90s),
  stepTimeout(archStep, 90s), stepTimeout(codeStep, 120s),
  stepTimeout(reviewStep, 90s), stepTimeout(verifyStep, 90s),
  stepTimeout(integrateStep, 90s).

- 1 Workflow SignoffWorkflow, one instance per ticketId. Steps:
  requestStep -> waitStep -> finalizeStep. requestStep: create a SignoffEntity
  record and call BlackboardEntity.awaitSignoff. waitStep: pauses; resumed
  externally by BoardEndpoint when a decision arrives. finalizeStep: read the
  decision; if approved call BlackboardEntity.approve (stage -> MERGED) and
  TicketEntity.complete; if rejected call BlackboardEntity.reject
  (stage -> IN_REVIEW) and re-invoke ControllerWorkflow from reviewStep.

- 3 EventSourcedEntities:
  * BlackboardEntity holding Blackboard state. Commands: receiveTicket, writePlan,
    writeArch, writeBackendCode, writeFrontendCode, writeReview, writeSecurity,
    writeTests, writeDocs, writeIntegrationPlan, awaitSignoff, approve, reject,
    alertStuck, getBoard. Before persisting any BackendCodeWritten or
    FrontendCodeWritten event, run the CodeValidator (control G1): if validation
    fails, return an error reply instead of emitting the event. BlackboardStage
    enum: INTAKE, PLANNED, ARCHITECTED, CODED, REVIEWED, VERIFIED,
    INTEGRATION_PLANNED, AWAITING_SIGNOFF, IN_REVIEW, MERGED.
    Events: TicketReceived, PlanWritten, ArchWritten, BackendCodeWritten,
    FrontendCodeWritten, ReviewWritten, SecurityWritten, TestsWritten, DocsWritten,
    IntegrationPlanWritten, SignoffRequested, SignoffApproved, SignoffRejected,
    BoardStuckAlerted.
  * TicketEntity holding Ticket state. Commands: createTicket, start, complete,
    reject, getTicket. TicketStatus enum: OPEN, IN_PROGRESS, DONE, REJECTED.
    Events: TicketCreated, TicketStarted, TicketCompleted, TicketRejected.
    emptyState() returns Ticket.initial("") with NO commandContext() reference.
  * SignoffEntity, id = ticketId. Commands: createRequest, recordDecision,
    getSignoff. Events: SignoffRequestCreated, SignoffDecisionRecorded.
    State: SignoffRequest + Optional<SignoffDecision>.

- 1 EventSourcedEntity TicketQueue, single instance "default". Command
  submitTicket(TicketBrief) emitting TicketSubmitted{ticketId, title, description,
  submittedBy, submittedAt}.

- 1 View BlackboardView with row type BoardRow (mirrors Blackboard minus the heavy
  CodeFile contents — keep artifact summaries and file counts only; no inline
  file contents in the view row). Table updater consumes BlackboardEntity events.
  ONE query getAllBoards SELECT * AS boards FROM blackboard_view. No WHERE filter
  on the stage enum column (Akka cannot auto-index enum columns — Lesson 2);
  callers filter client-side.

- 1 Consumer TicketConsumer subscribed to TicketQueue events; on TicketSubmitted
  calls TicketEntity.createTicket then starts a ControllerWorkflow with the
  ticketId as the workflow id.

- 2 TimedActions:
  * TicketSimulator — every 90s, reads the next line from
    src/main/resources/sample-events/ticket-briefs.jsonl and calls
    TicketQueue.submitTicket. Wraps when the file is exhausted.
  * StuckBoardMonitor — every 60s, queries BlackboardView.getAllBoards, finds
    boards in a non-terminal stage whose createdAt (or last event) is older than
    5 minutes, and calls BlackboardEntity.alertStuck.

- 2 HttpEndpoints:
  * BoardEndpoint at /api with POST /tickets, GET /tickets/{id},
    GET /boards, GET /boards/{ticketId}, GET /boards/sse,
    POST /signoff/{ticketId}, GET /signoff/{ticketId}, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 service-setup Bootstrap: schedule TicketSimulator and StuckBoardMonitor.

Companion files:
- BoardTasks.java declaring nine Task<R> constants: PLAN (resultConformsTo
  TaskBreakdown), ARCH (resultConformsTo ArchDecision), BACKEND_CODE
  (resultConformsTo CodeArtifact), FRONTEND_CODE (resultConformsTo CodeArtifact),
  REVIEW (resultConformsTo ReviewFindings), SECURITY (resultConformsTo
  SecurityFindings), TEST (resultConformsTo TestResults), DOC
  (resultConformsTo DocArtifact), INTEGRATE (resultConformsTo IntegrationPlan).
- Domain records TicketBrief, TaskItem, TaskBreakdown, ArchDecision, CodeFile,
  CodeArtifact, ReviewFinding, ReviewFindings, SecurityFinding, SecurityFindings,
  TestCase, TestResults, DocArtifact, IntegrationPlan, SignoffRequest,
  SignoffDecision in domain/.
- CodeValidator.java (deterministic, in application/) — a pure function over a
  CodeArtifact returning a ValidationResult: passes when every CodeFile has a
  non-blank path within a layer-scoped workspace, the content is non-empty, no
  file contains an unimplemented stub ("TODO" or
  "throw new UnsupportedOperationException"), and the path matches the layer
  prefix (backend/ or frontend/). This is the guardrail (G1); it is NOT an LLM
  call. The BlackboardEntity command handler calls it before emitting
  BackendCodeWritten or FrontendCodeWritten.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9421 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/ticket-briefs.jsonl with 6 canned ticket
  briefs (each a small self-contained engineering task — e.g., a rate-limiter
  API, a CSV export service, an auth middleware, a notification dispatcher, a
  config hot-reload module, a metrics aggregator).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 2 controls (G1 before-state-write
  guardrail, H1 hitl application signoff) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root.
- Nine prompt files in prompts/.
- README.md at the project root.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4
  mermaid diagrams + click-to-expand component table with syntax-highlighted Java
  snippets), Risk Survey, Eval Matrix, App UI (ticket form + stage pipeline per
  board + signoff panel + stuck alert banner). Browser title exactly:
  <title>Akka Sample: Blackboard SWE Coordination</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate MockModelProvider returning
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with a per-agent dispatch on the Task<R> id.
  Each branch reads a JSON file from src/main/resources/mock-responses/
  <agent-name>.json, picks one entry pseudo-randomly per call, and deserialises
  into the agent's typed return.
- Per-agent mock-response shapes:
    planner.json — 4 TaskBreakdown entries, each with 3-5 TaskItem records
      covering layers (backend, frontend, infra, shared).
    architect.json — 3 ArchDecision entries with plausible approach, components,
      constraints.
    backend-dev.json — 4 CodeArtifact entries (layer "backend"), 1-3 CodeFiles
      each. Include 1 entry with a "TODO" stub so the guardrail fires in tests.
    frontend-dev.json — 4 CodeArtifact entries (layer "frontend"), 1-3 CodeFiles
      each.
    reviewer.json — 4 ReviewFindings entries, most with approvedForTesting true;
      1 with a blocking finding.
    security-analyst.json — 3 SecurityFindings entries, most with clearToMerge
      true; 1 with a high-severity finding.
    tester.json — 4 TestResults entries, most with allPassed true; 1 failing.
    doc-writer.json — 3 DocArtifact entries with concise overview, apiSections,
      deploymentNotes.
    integration-planner.json — 3 IntegrationPlan entries with steps and
      rollbackSteps.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each agent
  has its companion BoardTasks Task<R> constant (Lesson 7).
- View has no WHERE filter on the stage enum column; filter client-side
  (Lesson 2).
- Workflow steps that call agents set explicit stepTimeouts (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9421 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26).
- UI is a single self-contained static-resources/index.html — no ui/ folder,
  no package.json, no npm build (Lesson 17).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during generation.
- No forbidden words in user-facing text (Lessons 21, 22, 23).
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
