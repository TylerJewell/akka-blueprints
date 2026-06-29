# SPEC — swarm-pattern

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Swarm Pattern.
**One-line pitch:** Submit a job brief; a coordinator agent splits it into work items on a shared list, and several worker agents claim items on their own, process them, and signal each other when they need to coordinate.

## 2. What this blueprint demonstrates

The **team-shared-list** coordination pattern in its most general form: a decomposition workflow runs a coordinator agent that writes a list of work items onto a shared list, and a fixed roster of worker agents each runs its own durable loop that pulls open items off that list, claims them atomically, produces a result, and marks them done. No central dispatcher hands work out — the workers self-organise around the shared list, and the work-item entity's single-writer guarantee is what prevents two workers from claiming the same item.

The blueprint demonstrates two governance mechanisms that a self-organising swarm needs: a **quality gate** that a result must pass before an item counts as done, and an **operator pause** that freezes the whole swarm.

## 3. User-facing flows

The user opens the App UI tab and submits a job via the form (a title plus a free-text description).

1. The system logs the submission on `SubmissionLog` and a `Consumer` starts a `DecompositionWorkflow` for the job.
2. The `Coordinator` agent decomposes the brief into a `WorkPlan` — a list of work items, each with a title, a description, and the titles of items it depends on. The workflow writes one `WorkItemEntity` per item (status `OPEN`) and moves the job to `READY`.
3. The worker roster (`worker-1`, `worker-2`, `worker-3`) is already running — each worker is a `WorkerWorkflow` that polls the shared `WorkListView` for an `OPEN` item whose dependencies are all `DONE`.
4. A worker claims an eligible item. The claim is an atomic command on that item's `WorkItemEntity`; if two workers race for the same item, one wins and the other goes back to polling.
5. The winning worker runs the `Worker` agent, which produces a `WorkResult` (a list of output entries and a summary). The result runs through a quality gate. On pass, the item becomes `DONE`. On fail, the worker retries up to a fixed limit, then the item becomes `STALLED`.
6. If the worker reports it needs another worker's output, it raises a `RelayRequest`. The item goes `STALLED`, a `RelayMessage` lands in the target worker's `RelayMailbox`, and the worker who can answer posts a reply that unblocks the waiting item.
7. When every item in a job is `DONE`, the job moves to `FINISHED`.

A `JobSimulator` (TimedAction) drips a sample job brief every 60 seconds so the App UI is not empty when first loaded. A `StalledItemMonitor` (TimedAction) releases any item that has been claimed but idle for more than two minutes, returning it to `OPEN` so another worker can pick it up.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `Coordinator` | `AutonomousAgent` | Decomposes a job brief into a list of work items with dependencies. | `DecompositionWorkflow` | returns typed `WorkPlan` to workflow |
| `Worker` | `AutonomousAgent` | Claims and processes one work item; returns a `WorkResult`; may raise a `RelayRequest`. Run as several instances (`worker-1`, `worker-2`, `worker-3`). | `WorkerWorkflow` | returns typed result to workflow |
| `DecompositionWorkflow` | `Workflow` | Runs `Coordinator`, creates one `WorkItemEntity` per item, moves the job to `READY`. | `JobRequestConsumer` | `Coordinator`, `WorkItemEntity`, `JobEntity` |
| `WorkerWorkflow` | `Workflow` | Per-worker durable loop: poll → claim → process → quality gate → done/stalled/relay. | `Bootstrap` (one instance per worker id) | `WorkItemEntity`, `Worker`, `RelayMailbox`, `SwarmControl` |
| `JobEntity` | `EventSourcedEntity` | Job lifecycle: brief, item ids, status. | `DecompositionWorkflow`, `JobCompletionMonitor` | read via endpoint |
| `WorkItemEntity` | `EventSourcedEntity` | One per work item. Atomic `claim`; full item lifecycle. | `DecompositionWorkflow`, `WorkerWorkflow`, `StalledItemMonitor` | `WorkListView` |
| `RelayMailbox` | `EventSourcedEntity` | One per worker. Inbound relay messages and replies. | `WorkerWorkflow` | `SwarmEndpoint` |
| `SubmissionLog` | `EventSourcedEntity` | Single instance; logs each submitted job for audit. | `SwarmEndpoint`, `JobSimulator` | `JobRequestConsumer` |
| `SwarmControl` | `KeyValueEntity` | Operator pause flag for the whole swarm. | `SwarmEndpoint` | `WorkerWorkflow` |
| `WorkListView` | `View` | The shared work-item read model the workers poll and the UI streams. | `WorkItemEntity` events | `WorkerWorkflow`, `SwarmEndpoint` |
| `JobRequestConsumer` | `Consumer` | Listens to `SubmissionLog` events; starts a `DecompositionWorkflow` per submission. | `SubmissionLog` events | `DecompositionWorkflow` |
| `JobSimulator` | `TimedAction` | Drips a sample job brief every 60 s. | scheduler | `SubmissionLog` |
| `StalledItemMonitor` | `TimedAction` | Every 30 s, releases items claimed but idle > 2 min back to `OPEN`. | scheduler | `WorkItemEntity` |
| `SwarmEndpoint` | `HttpEndpoint` | `/api/*` — submit, list items, get job, relay reply, pause/resume, SSE. | — | `SubmissionLog`, `WorkListView`, `JobEntity`, `RelayMailbox`, `SwarmControl` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions; starts one `WorkerWorkflow` per worker id. | — | `WorkerWorkflow`, scheduler |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record JobBrief(String jobId, String title, String description, String submittedBy) {}

record ItemSpec(String title, String description, List<String> dependsOn) {}
record WorkPlan(String planSummary, List<ItemSpec> items) {}

record OutputEntry(String key, String value) {}
record RelayRequest(String toWorker, String question) {}
record WorkResult(String itemId, List<OutputEntry> outputs, String summary,
                  Optional<RelayRequest> relayRequest) {}

record QualityReport(boolean passed, List<String> issues, Instant ranAt) {}
record RelayMessage(String messageId, String fromWorker, String toWorker, String itemId,
                    String question, Instant sentAt, Optional<String> reply,
                    Optional<Instant> repliedAt) {}
```

### Entity state — `WorkItem` (state of `WorkItemEntity`, basis of the view row)

```java
record WorkItem(
    String itemId,
    String jobId,
    String title,
    String description,
    List<String> dependsOn,
    WorkItemStatus status,
    Optional<String>  claimedBy,
    Optional<Instant> claimedAt,
    Optional<String>  resultSummary,
    Optional<Integer> outputCount,
    Optional<QualityReport> qualityReport,
    Optional<String>  stalledReason,
    Optional<Instant> completedAt,
    Instant createdAt
) {}

enum WorkItemStatus { OPEN, CLAIMED, IN_PROGRESS, IN_REVIEW, DONE, STALLED }
```

### Entity state — `Job` (state of `JobEntity`)

```java
record Job(
    String jobId,
    String title,
    String description,
    String submittedBy,
    JobStatus status,
    List<String> itemIds,
    Optional<String> planSummary,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus { PLANNING, READY, IN_PROGRESS, FINISHED }
```

### Events

`WorkItemEntity`: `WorkItemCreated`, `WorkItemClaimed`, `WorkItemStarted`, `ResultSubmitted`, `QualityPassed`, `QualityFailed`, `WorkItemStalled`, `WorkItemReleased`, `WorkItemCompleted`.
`JobEntity`: `JobCreated`, `JobPlanned`, `JobStarted`, `JobFinished`.
`RelayMailbox`: `RelayPosted`, `RelayReplied`.
`SubmissionLog`: `JobSubmitted`.
`SwarmControl` is a `KeyValueEntity` — state transitions are command-driven, no events.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ title, description, submittedBy? }` → `{ jobId }`. Logs the submission and starts decomposition.
- `GET /api/jobs/{id}` — one job with its item ids and status.
- `GET /api/items` — list all work items. Optional `?jobId=...` and `?status=...`, filtered client-side.
- `GET /api/items/sse` — server-sent events stream of every item change.
- `GET /api/mailbox/{workerId}` — relay messages for a worker.
- `POST /api/mailbox/{workerId}/messages/{messageId}/reply` — body `{ reply }`. Posts a reply that unblocks the waiting item.
- `POST /api/control/pause` — body `{ reason, by }`. Sets the operator pause flag.
- `POST /api/control/resume` — clears the pause flag.
- `GET /api/control` — current pause state.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Swarm Pattern</title>`.

- **Overview** — eyebrow "Overview" + headline "Swarm Pattern"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, item state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a job submission form, a live board grouped by status column (Open / Claimed / In progress / In review / Done / Stalled), per-item cards showing the claiming worker and result summary, a relay-mailbox panel with reply boxes, and a Pause / Resume control.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. This baseline carries no governance controls; `eval-matrix.yaml` is a structural placeholder with an empty `controls` list and `simplified_view` list. The generated system still wires the quality gate and the operator pause as structural mechanisms — they are demonstrated but not formally enumerated as controls in this baseline.

## 9. Agent prompts

- `Coordinator` → `prompts/coordinator.md`. Decomposes a job brief into a dependency-ordered work-item list.
- `Worker` → `prompts/worker.md`. Claims an item, produces a result, raises a relay request when it needs another worker's output.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a job; the coordinator writes items to the list; workers claim and complete them; the job reaches `FINISHED`. The list updates live via SSE.
2. **J2** — Two workers poll at the same instant; each item ends up claimed by exactly one worker (no double-claim).
3. **J3** — A worker raises a relay request; the item goes `STALLED`, a message lands in the target's mailbox, and a reply unblocks it.
4. **J4** — The operator pauses the swarm; claiming pauses; resume continues the work.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named swarm-pattern demonstrating the
team-shared-list coordination pattern in the general domain.
Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact team-shared-list-general-swarm-pattern.
Java package io.akka.samples.swarmpattern. Akka 3.6.0. HTTP port 9405.

Components to wire (exactly):
- 2 AutonomousAgents:
  * Coordinator — definition() with capability(SwarmTasks.of(DECOMPOSE)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/coordinator.md.
    Returns WorkPlan{planSummary, items: List<ItemSpec{title, description,
    dependsOn: List<String>}>}.
  * Worker — definition() with capability(SwarmTasks.of(PROCESS)
    .maxIterationsPerTask(3)). System prompt from prompts/worker.md. Returns
    WorkResult{itemId, outputs: List<OutputEntry{key, value}>, summary,
    relayRequest: Optional<RelayRequest{toWorker, question}>}. Run as several
    runtime instances addressed by instanceId (worker-1, worker-2, worker-3) —
    ONE agent class, several instance ids; never generate three agent classes.

- 1 Workflow DecompositionWorkflow with steps decomposeStep -> createItemsStep ->
  readyStep. decomposeStep calls forAutonomousAgent(Coordinator.class, jobId)
  .runSingleTask(DECOMPOSE.instructions(brief)) then forTask(taskId)
  .result(DECOMPOSE). createItemsStep writes one WorkItemEntity per ItemSpec with a
  deterministic itemId = jobId + "-i" + index and calls
  JobEntity.recordPlan(itemIds, planSummary). readyStep ends the workflow.
  Override settings() with stepTimeout(decomposeStep, 90s).

- 1 Workflow WorkerWorkflow, one durable instance per worker id. State carries
  workerId and the currently-claimed itemId (Optional). Steps:
  pollStep -> claimStep -> workStep -> qualityGateStep -> finishStep, with a
  self-scheduled resume timer for idling. pollStep: read SwarmControl; if paused,
  schedule a 5s resume timer and pause. Otherwise query
  WorkListView.getAllItems, pick the oldest OPEN item whose dependsOn titles
  are all DONE; if none, schedule a 5s resume timer and pause; else go to
  claimStep. claimStep: call WorkItemEntity.claim(workerId); if the entity
  reports the item is no longer OPEN (lost the race), go back to pollStep; else
  WorkItemEntity.start and go to workStep. workStep (stepTimeout 90s): call
  forAutonomousAgent(Worker.class, workerId).runSingleTask(
  PROCESS.instructions(item)). On a RelayRequest in the result, call
  RelayMailbox(toWorker).post(message) and WorkItemEntity.stall("waiting on
  relay: " + toWorker), then go to pollStep. Otherwise call
  WorkItemEntity.submitResult(result) and go to qualityGateStep.
  qualityGateStep: run the deterministic QualityChecker over the result; on pass
  call WorkItemEntity.passQuality(report) (-> DONE) and go to finishStep; on fail
  up to 2 retries call WorkItemEntity.failQuality(report) and loop workStep; after
  retries exhausted call WorkItemEntity.stall("quality failed") and go to pollStep.
  finishStep: clear the claimed item and go to pollStep.

- 4 EventSourcedEntities:
  * WorkItemEntity holding WorkItem state. Commands: createItem, claim, start,
    submitResult, passQuality, failQuality, stall, release, getItem. claim emits
    WorkItemClaimed ONLY when status==OPEN; otherwise it is a no-op that returns
    the current WorkItem so the caller can detect the lost race. WorkItemStatus
    enum: OPEN, CLAIMED, IN_PROGRESS, IN_REVIEW, DONE, STALLED. Events:
    WorkItemCreated, WorkItemClaimed, WorkItemStarted, ResultSubmitted,
    QualityPassed, QualityFailed, WorkItemStalled, WorkItemReleased,
    WorkItemCompleted. emptyState() returns WorkItem.initial("", "") with NO
    commandContext() reference.
  * JobEntity holding Job state. Commands: createJob, recordPlan, start, finish,
    getJob. JobStatus enum: PLANNING, READY, IN_PROGRESS, FINISHED. Events:
    JobCreated, JobPlanned, JobStarted, JobFinished. emptyState() returns
    Job.initial("") with NO commandContext() reference.
  * RelayMailbox, id = workerId. Commands: post(RelayMessage), reply(messageId,
    text), getMailbox. Events: RelayPosted, RelayReplied. State is a
    List<RelayMessage>.
  * SubmissionLog, single instance "default". Command submitJob(JobBrief)
    emitting JobSubmitted{jobId, title, description, submittedBy, submittedAt}.

- 1 KeyValueEntity SwarmControl, single instance "default", holding
  {paused: boolean, pausedReason: Optional<String>, pausedBy: Optional<String>,
  pausedAt: Optional<Instant>}. Commands: pause(reason, by), resume, getControl.

- 1 View WorkListView with row type WorkItemRow (mirrors WorkItem minus the full
  OutputEntry list — keep resultSummary and outputCount only). Table updater
  consumes WorkItemEntity events. ONE query getAllItems SELECT * AS items FROM
  work_list. No WHERE status filter (Akka cannot auto-index enum columns —
  Lesson 2); callers filter client-side.

- 1 Consumer JobRequestConsumer subscribed to SubmissionLog events; on
  JobSubmitted calls JobEntity.createJob then starts a DecompositionWorkflow with
  the jobId as the workflow id.

- 2 TimedActions:
  * JobSimulator — every 60s, reads the next line from
    src/main/resources/sample-events/job-briefs.jsonl and calls
    SubmissionLog.submitJob. Stops cycling when the file is exhausted (wraps).
  * StalledItemMonitor — every 30s, queries WorkListView.getAllItems, finds
    items in CLAIMED or IN_PROGRESS whose claimedAt is older than 2 minutes,
    and calls WorkItemEntity.release (-> OPEN, clears claimedBy).

- 2 HttpEndpoints:
  * SwarmEndpoint at /api with POST /jobs, GET /jobs/{id},
    GET /items (client-side filter by jobId/status), GET /items/sse,
    GET /mailbox/{workerId}, POST /mailbox/{workerId}/messages/{id}/reply,
    POST /control/pause, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 service-setup Bootstrap: schedule JobSimulator and StalledItemMonitor;
  start one WorkerWorkflow instance per worker id (worker-1, worker-2, worker-3).

Companion files:
- SwarmTasks.java declaring two Task<R> constants: DECOMPOSE (resultConformsTo
  WorkPlan) and PROCESS (resultConformsTo WorkResult).
- Domain records JobBrief, ItemSpec, WorkPlan, OutputEntry, RelayRequest,
  WorkResult, QualityReport, RelayMessage in domain/.
- QualityChecker.java (deterministic, in application/) — a pure function over a
  WorkResult returning a QualityReport: passes when every OutputEntry has a
  non-blank key and value, the summary is non-empty, and no value contains a
  literal "TODO" or "PLACEHOLDER"; otherwise fails with one issue string per
  offending entry.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9405 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also a worker-roster setting
  swarm.workers = ["worker-1","worker-2","worker-3"] read by Bootstrap.
- src/main/resources/sample-events/job-briefs.jsonl with 6 canned job briefs
  (each a small, self-contained processing job — e.g., summarising a set of
  articles, classifying customer feedback items, extracting entities from a
  document batch, tagging a product catalog, scoring survey responses, ranking
  search results).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with an empty controls list and an empty
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md and prompts/worker.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Swarm Pattern",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid
  diagrams + click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sections; unanswered .qb opacity 0.45), Eval Matrix (5-column
  table — empty body since no controls), App UI (job form + board with one column
  per WorkItemStatus, a relay-mailbox panel with reply boxes, and a Pause/Resume
  control). Browser title exactly: <title>Akka Sample: Swarm Pattern</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via
  the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory only.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch on agent class name.
  Each branch reads a JSON file from src/main/resources/mock-responses/:
    coordinator.json — 4-6 WorkPlan entries. Each has a planSummary and 3-5
      ItemSpec items with believable processing tasks (e.g., "Load article batch",
      "Extract named entities", "Classify sentiment", "Aggregate results"), with
      dependsOn referencing earlier item titles.
    worker.json — 4-6 WorkResult entries. Most have 2-4 OutputEntry items with
      non-blank, PLACEHOLDER-free key/value pairs and a one-sentence summary and
      an empty relayRequest. Include 1 entry whose relayRequest is present
      (toWorker "worker-2", a concrete question) to exercise relay coordination,
      and include 1 entry whose value contains a literal "TODO" so the quality
      gate fails deterministically.
- MockModelProvider.seedFor(itemId) makes selection deterministic per item id.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion SwarmTasks Task<R> constants (Lesson 7).
- View has no WHERE filter on the enum status column; filter client-side (Lesson 2).
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9405 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS overrides
  AND theme variables from Lesson 24.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26).
- UI is a single self-contained static-resources/index.html — no ui/ folder, no
  package.json, no npm build (Lesson 17).
- No forbidden words in user-facing text (Lessons 21, 22, 23).
- Constraints list references Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24,
  25, 26 by number.
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
