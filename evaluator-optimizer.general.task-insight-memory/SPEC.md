# SPEC — task-insight-memory

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Task-Centric Memory.
**One-line pitch:** Submit a task; an executor agent retrieves relevant prior insights, runs the task, and an evaluator agent scores the outcome; verified results are persisted back into the memory store so every future execution of that task type benefits from accumulated experience.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern applied to memory accumulation: a `MemoryWorkflow` orchestrates an `ExecutorAgent` (generates a result using retrieved insights) and an `EvaluatorAgent` (scores the result against acceptance criteria), with the evaluator's verdict gating whether the result is distilled into a new insight. The blueprint also demonstrates three governance mechanisms — an **eval-event** that records every outcome before persisting insight (verified-experience path), a **PII sanitizer** that scrubs each candidate insight before it reaches the memory store, and a **periodic drift watch** that monitors the accumulated insight distribution for fairness and quality drift over time.

## 3. User-facing flows

The user opens the App UI tab and submits a task (a task type identifier plus a free-text description).

1. The system creates a `TaskRecord` in `PENDING` and starts a `MemoryWorkflow`.
2. The workflow queries `MemoryView` for insights keyed to this task type (up to `maxInsights` = 5 most-recent verified insights).
3. The `ExecutorAgent` receives the task plus retrieved insights and produces a `TaskResult` with an answer and a self-assessed confidence score.
4. The `EvaluatorAgent` scores the result against the task's acceptance criteria and returns either `VERIFIED` with a rationale, or `REJECTED` with a typed `EvalNotes` payload (two or three specific bullets).
5. On `VERIFIED` with confidence >= threshold, the workflow sanitizes the result's key findings into a candidate `InsightCandidate`, runs the PII sanitizer step, and emits `InsightPersisted` on `MemoryEntity`.
6. On `REJECTED` or below-threshold confidence, the workflow records the attempt, the eval verdict, and the reason on `TaskEntity` and ends with `TaskResult` status `FAILED`.
7. If the workflow encounters an unrecoverable agent failure it emits `TaskFailed` with a structured reason; the memory store is never written in a partial state.

A `TaskSimulator` (TimedAction) drips a canned task every 60 seconds so the App UI is not empty when first loaded.

A `DriftWatcher` (TimedAction) fires every 5 minutes, queries `MemoryView`, and emits a `DriftAssessmentRecorded` event when the distribution of insight task types or confidence scores deviates significantly from the baseline snapshot established at startup.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ExecutorAgent` | `AutonomousAgent` | Retrieves relevant insights, executes the task, returns `TaskResult`. | `MemoryWorkflow` | returns `TaskResult` to workflow |
| `EvaluatorAgent` | `AutonomousAgent` | Scores a `TaskResult` against task acceptance criteria; returns `VERIFIED` or `REJECTED` with notes. | `MemoryWorkflow` | returns `EvalVerdict` to workflow |
| `MemoryWorkflow` | `Workflow` | Orchestrates retrieve → execute → evaluate → sanitize → persist loop. | `MemoryEndpoint`, `TaskConsumer` | `TaskEntity`, `MemoryEntity` |
| `MemoryEntity` | `EventSourcedEntity` | Holds the insight store: all `MemoryInsight` records keyed by task type, provenance, and text. | `MemoryWorkflow`, `MemoryEndpoint` | `MemoryView` |
| `TaskEntity` | `EventSourcedEntity` | Holds per-task lifecycle, every result attempt, and every eval verdict. | `MemoryWorkflow` | `TaskView` |
| `TaskQueue` | `EventSourcedEntity` | Logs each submitted task for replay and audit. | `MemoryEndpoint`, `TaskSimulator` | `TaskConsumer` |
| `MemoryView` | `View` | Read model of all insights, queryable by task type. | `MemoryEntity` events | `MemoryEndpoint`, `MemoryWorkflow` |
| `TaskView` | `View` | List-of-tasks read model. | `TaskEntity` events | `MemoryEndpoint` |
| `TaskConsumer` | `Consumer` | Subscribes to `TaskQueue` events; starts a `MemoryWorkflow` per submission. | `TaskQueue` events | `MemoryWorkflow` |
| `TaskSimulator` | `TimedAction` | Drips a sample task every 60 s from `sample-events/task-samples.jsonl`. | scheduler | `TaskQueue` |
| `DriftWatcher` | `TimedAction` | Every 5 min, queries `MemoryView`, computes distribution metrics, emits `DriftAssessmentRecorded` if deviation exceeds threshold. | scheduler | `MemoryEntity` |
| `MemoryEndpoint` | `HttpEndpoint` | `/api/tasks/*` and `/api/memory/*` — submit, get, list, SSE, insight CRUD; plus `/api/metadata/*`. | — | `TaskView`, `MemoryView`, `TaskQueue`, `TaskEntity`, `MemoryEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record TaskSpec(String taskType, String description, String acceptanceCriteria, String requestedBy) {}

record RetrievedInsight(String insightId, String taskType, String text, double confidence, InsightProvenance provenance) {}

record TaskResult(String answer, double confidence, List<String> keyFindings, Instant completedAt) {}

record EvalNotes(List<String> bullets, String overallRationale) {}

record EvalVerdict(EvalOutcome outcome, EvalNotes notes, double qualityScore, Instant evaluatedAt) {}

record InsightCandidate(String taskType, String sanitizedText, double confidence, InsightProvenance provenance) {}

record MemoryInsight(
    String insightId,
    String taskType,
    String text,
    double confidence,
    InsightProvenance provenance,
    Instant persistedAt,
    Optional<Instant> supersededAt
) {}

record TaskRecord(
    String taskId,
    String taskType,
    String description,
    String acceptanceCriteria,
    TaskStatus status,
    List<RetrievedInsight> retrievedInsights,
    Optional<TaskResult> result,
    Optional<EvalVerdict> evalVerdict,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus { PENDING, EXECUTING, EVALUATED, VERIFIED, FAILED }

enum EvalOutcome { VERIFIED, REJECTED }

enum InsightProvenance { CORRECTION, DEMONSTRATION, VERIFIED_EXPERIENCE }
```

### Events (on `TaskEntity`)

`TaskCreated`, `TaskExecutionStarted`, `TaskResultRecorded`, `TaskEvalVerdictRecorded`, `TaskVerified`, `TaskFailed`.

### Events (on `MemoryEntity`)

`InsightPersisted`, `InsightSuperseded`, `CorrectionApplied`, `DemonstrationAdded`, `DriftAssessmentRecorded`.

See `reference/data-model.md` for the full tables.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ taskType, description, acceptanceCriteria?, requestedBy? }` → `{ taskId }`. Starts a workflow.
- `GET /api/tasks` — list all tasks. Optional `?status=PENDING|EXECUTING|EVALUATED|VERIFIED|FAILED`.
- `GET /api/tasks/{id}` — one task (including result, eval verdict, retrieved insights).
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `GET /api/memory` — list all insights. Optional `?taskType=<string>`.
- `GET /api/memory/{insightId}` — one insight.
- `POST /api/memory/correction` — body `{ taskType, correctedText, supersedes? }` → `{ insightId }`. Operator correction path.
- `POST /api/memory/demonstration` — body `{ taskType, demonstrationText }` → `{ insightId }`. Admin demonstration path.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Task-Centric Memory"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, sanitizer = orange, eval-periodic = blue).
- **App UI** — form to submit a task, live list of tasks with status pills, click-to-expand per-task detail showing retrieved insights, the task result, the eval verdict, and the memory insight (if persisted). A separate Memory panel lists all insights grouped by task type.

Browser title: `<title>Akka Sample: Task-Centric Memory</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event** (`on-decision-eval`): every task outcome is evaluated by `EvaluatorAgent` before any insight is persisted. The eval record — `EvalVerdict{outcome, qualityScore, notes}` — is written as `TaskEvalVerdictRecorded` on `TaskEntity`. `REJECTED` results never reach the memory store. Enforcement: non-blocking (the eval is mandatory but does not block the HTTP response to the submitter).
- **S1 — sanitizer** (`pii`): before an `InsightCandidate` is written to `MemoryEntity`, the workflow runs a deterministic PII scrub step: e-mail addresses, phone numbers, and name-shaped tokens are replaced with `[REDACTED]`. The scrubbed candidate is what gets persisted; the raw result is never stored in `MemoryEntity`. Enforcement: non-blocking (advisory on each persist; a configurable strict mode can make it blocking).
- **P1 — eval-periodic** (`drift-fairness-watch`): `DriftWatcher` (TimedAction) fires every 5 minutes. It queries `MemoryView`, computes the distribution of insight task types and the moving average confidence score, and compares against a baseline snapshot. If type concentration exceeds 80 % for a single task type or average confidence drops below the floor, `DriftAssessmentRecorded` is emitted with a `DriftReason` payload. Enforcement: system-level (the event is observable but does not auto-prune the store).

## 9. Agent prompts

- `ExecutorAgent` → `prompts/executor.md`. Retrieves and weighs prior insights; executes the task; returns a `TaskResult` with a confidence score.
- `EvaluatorAgent` → `prompts/evaluator.md`. Scores a `TaskResult` against the task's acceptance criteria; returns `VERIFIED` or `REJECTED` with structured notes.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — verified persistence** — Submit a task; it progresses `PENDING` → `EXECUTING` → `EVALUATED` → `VERIFIED`; a new insight appears in the memory store; the App UI shows the retrieved insights, the result, and the eval verdict.
2. **J2 — rejection no-write** — Submit a task the evaluator is configured to reject (test mode); status reaches `FAILED`; no new insight appears in `MemoryView`; the task's detail view shows the rejection notes.
3. **J3 — correction path** — `POST /api/memory/correction` overwrites an existing insight; the next task execution of that type retrieves the corrected text.
4. **J4 — drift alert** — After accumulating several insights skewed toward a single task type, the `DriftWatcher` emits `DriftAssessmentRecorded`; the event surfaces in the App UI's memory timeline with the `DriftReason` and triggering metrics.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named task-insight-memory demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-task-insight-memory.
Java package io.akka.samples.taskcentricmemory. Akka 3.6.0. HTTP port 9663.

Components to wire (exactly):
- 2 AutonomousAgents:
  * ExecutorAgent — definition() with
    capability(TaskAcceptance.of(EXECUTE_TASK).maxIterationsPerTask(4)).
    System prompt loaded from prompts/executor.md. Returns TaskResult{answer,
    confidence, keyFindings, completedAt}. Input is (taskType, description,
    acceptanceCriteria, List<RetrievedInsight> priorInsights).
  * EvaluatorAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE_RESULT).maxIterationsPerTask(2)).
    System prompt loaded from prompts/evaluator.md. Returns EvalVerdict{outcome,
    notes, qualityScore, evaluatedAt} where outcome is EvalOutcome enum
    (VERIFIED | REJECTED).

- 1 Workflow MemoryWorkflow with steps:
    startStep -> retrieveStep -> executeStep -> evaluateStep ->
    [outcome VERIFIED AND confidence >= threshold?
       sanitizeStep -> persistStep -> END
     : failStep -> END].
  retrieveStep calls MemoryView.getInsightsByTaskType(taskType, maxInsights=5)
    — a pure-function step, no LLM call.
  executeStep calls forAutonomousAgent(ExecutorAgent.class, taskId)
    .runSingleTask(EXECUTE_TASK) then forTask(taskId).result(EXECUTE_TASK).
  evaluateStep calls forAutonomousAgent(EvaluatorAgent.class, taskId)
    .runSingleTask(EVALUATE_RESULT).
  sanitizeStep is a pure-function step (no LLM call): applies regex-based
    PII scrub (email, phone, name-shaped tokens → [REDACTED]) to each
    keyFinding. Emits SanitizerApplied with redactionCount.
  persistStep emits InsightPersisted on MemoryEntity with InsightCandidate
    {taskType, sanitizedText (joined keyFindings), confidence, provenance=VERIFIED_EXPERIENCE}.
  failStep emits TaskFailed with structured failureReason.
  Override settings() with stepTimeout(60s) on executeStep and evaluateStep,
    and defaultStepRecovery(maxRetries(2).failoverTo(failStep)).
  Confidence threshold read from task-insight-memory.executor.min-confidence
    (default 0.70).

- 2 EventSourcedEntities:
  * MemoryEntity holding state MemoryStore{storeId, List<MemoryInsight> insights,
    SnapshotMetrics baselineMetrics, Instant lastAssessedAt}. Events:
    InsightPersisted, InsightSuperseded, CorrectionApplied, DemonstrationAdded,
    DriftAssessmentRecorded. Commands: persistInsight, applyCorrection,
    addDemonstration, supersede, recordDriftAssessment, getStore.
  * TaskEntity holding state TaskRecord{taskId, taskType, description,
    acceptanceCriteria, TaskStatus status, List<RetrievedInsight> retrievedInsights,
    Optional<TaskResult> result, Optional<EvalVerdict> evalVerdict,
    Optional<String> failureReason, Instant createdAt, Optional<Instant> finishedAt}.
    Events: TaskCreated, TaskExecutionStarted, TaskResultRecorded,
    TaskEvalVerdictRecorded, TaskVerified, TaskFailed. Commands: createTask,
    startExecution, recordResult, recordEvalVerdict, verify, fail, getTask.
    emptyState() returns TaskRecord.initial("","",TaskStatus.PENDING) with no
    commandContext() reference. Event-applier wraps lifecycle fields with
    Optional.of(...).

- 1 EventSourcedEntity TaskQueue with command submitTask(taskType, description,
  acceptanceCriteria, requestedBy) emitting TaskSubmitted{taskId, taskType,
  description, acceptanceCriteria, requestedBy, submittedAt}.

- 2 Views:
  * MemoryView with row type InsightRow (mirrors MemoryInsight). TWO queries:
    getAllInsights SELECT * AS insights FROM memory_view (full store);
    getInsightsByTaskType SELECT * AS insights FROM memory_view WHERE taskType = :taskType.
  * TaskView with row type TaskRow (mirrors TaskRecord). ONE query:
    getAllTasks SELECT * AS tasks FROM task_view.
    No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer TaskConsumer subscribed to TaskQueue events; on TaskSubmitted
  starts a MemoryWorkflow with taskId as the workflow id.

- 3 TimedActions:
  * TaskSimulator — every 60s, reads next line from
    src/main/resources/sample-events/task-samples.jsonl and calls
    TaskQueue.submitTask.
  * DriftWatcher — every 5 min, queries MemoryView.getAllInsights, computes
    type-concentration ratio and moving-average confidence. If concentration
    > 0.80 OR avg confidence < task-insight-memory.drift.confidence-floor
    (default 0.55), calls MemoryEntity.recordDriftAssessment(reason, metrics).
    Idempotent on (assessmentMinute) — one assessment per 5-min window max.

- 2 HttpEndpoints:
  * MemoryEndpoint at /api with:
    POST /tasks, GET /tasks, GET /tasks/{id}, GET /tasks/sse,
    GET /memory, GET /memory/{insightId},
    POST /memory/correction, POST /memory/demonstration,
    and three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
    POST /tasks body is {taskType, description, acceptanceCriteria?,
    requestedBy?}; missing acceptanceCriteria defaults to "answer is accurate
    and complete", missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- MemoryTasks.java declaring two Task<R> constants: EXECUTE_TASK (resultConformsTo
  TaskResult), EVALUATE_RESULT (EvalVerdict).
- Domain records TaskSpec, RetrievedInsight, TaskResult, EvalNotes, EvalVerdict,
  InsightCandidate, MemoryInsight, TaskRecord, SnapshotMetrics, DriftReason;
  enums TaskStatus, EvalOutcome, InsightProvenance.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9663
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  canonical env vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}).
  Per-workflow config task-insight-memory.executor.min-confidence = 0.70,
  task-insight-memory.executor.max-insights = 5,
  task-insight-memory.drift.confidence-floor = 0.55, overridable by env var.
- src/main/resources/sample-events/task-samples.jsonl with 8 canned task lines,
  each shaped {"taskType":"...","description":"...","acceptanceCriteria":"..."}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (E1 eval-event
  on-decision-eval, S1 sanitizer pii, P1 eval-periodic drift-fairness-watch)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root pre-filling
  purpose.primary_function = task-execution-memory,
  decisions.authority_level = advisory, data.data_classes.pii = true,
  capabilities.content-generation = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/executor.md, prompts/evaluator.md loaded at agent startup.
- README.md at the project root: title "Akka Sample: Task-Centric Memory",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (task submit form + live task list with status pills,
  click-to-expand per-task detail, Memory panel listing all insights by task
  type). Browser title exactly:
  <title>Akka Sample: Task-Centric Memory</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  ModelProvider with per-agent dispatch on agent class name.
- Per-agent mock-response shapes:
    executor.json — 6 TaskResult entries. Four have confidence >= 0.70 and
      2-4 keyFindings describing the task domain. Two have confidence < 0.70
      to exercise the below-threshold path (J2 in user-journeys).
    evaluator.json — 6 EvalVerdict entries. Three return outcome=VERIFIED with
      qualityScore >= 0.80 and a one-sentence rationale. Three return
      outcome=REJECTED with qualityScore < 0.60 and an EvalNotes payload of
      two or three bullets naming specific shortfalls.
- A MockModelProvider.seedFor(taskId) helper makes selection deterministic
  per taskId across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. ExecutorAgent
  and EvaluatorAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a MemoryTasks companion declaring the two Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on TaskRecord is Optional<T>;
  the event-applier wraps values with Optional.of(...).
- Lesson 7: MemoryTasks.java is mandatory; generating ExecutorAgent or
  EvaluatorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9663, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK
  in narrative, marketing tone, competitor brand names) do not appear in
  any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND theme variables for state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records only
  ${?VAR_NAME} substitution; Bootstrap.java fails fast if the reference does
  not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. The DOM contains exactly five <section class="tab-panel">
  elements; removed panels are deleted from the HTML, not hidden with
  display:none.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
