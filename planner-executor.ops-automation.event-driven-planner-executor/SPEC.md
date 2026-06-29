# SPEC — event-driven-planner-executor

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Workflow Integration.
**One-line pitch:** Submit an automation job; an Orchestrator plans the work on a job ledger, dispatches each step to one of four executor agents (HTTP caller, queue publisher, database query, script runner), records the outcome on a step ledger, and replans when steps fail.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern in an operations-automation setting where each executor wraps a durable, retryable workflow step. The Orchestrator owns two ledgers — a **job ledger** (facts known, facts to discover, plan, current dispatch) and a **step ledger** (each step's attempt count, verdict, result, blockers). Each loop iteration the Orchestrator reads both ledgers, picks the next executor, and either continues, replans, halts, or completes. On three consecutive failures of the same step, or two consecutive replans without progress, the Orchestrator emits a terminal failure.

The blueprint also demonstrates two governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each executor invocation against an allow-list and a content policy before the step fires,
- a **deployer runtime-monitoring** surface (operator dashboard + manual halt control) that lets a human-on-the-loop pause new dispatches without killing in-flight steps.

## 3. User-facing flows

The user opens the App UI tab and submits a job via the form.

1. The system creates a `Job` record in `PLANNING` and starts a `JobWorkflow`.
2. The Orchestrator drafts a `JobLedger { facts, missing, plan, dispatch }` and emits `JobPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - Orchestrator reads both ledgers and proposes a `DispatchDecision { executor, step, rationale }`.
   - The **before-tool-call guardrail** vets the decision; on rejection the workflow records a `StepBlocked` entry on the step ledger and asks the Orchestrator to revise.
   - The chosen executor runs the step and returns a typed `StepResult`.
   - The **credential sanitizer** scrubs the result.
   - The workflow appends a `StepEntry { executor, step, attempt, verdict, scrubbedResult }` to the step ledger.
4. The Orchestrator decides on each loop tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`. After three consecutive failures on the same step or two consecutive replans without progress, the Orchestrator emits `FAIL`.
5. On `COMPLETE`, the Orchestrator produces a `JobReport { summary, evidence }` and emits `JobCompleted`. The Job moves to `COMPLETED`.
6. The operator can press **Halt new dispatches** in the dashboard at any time. The workflow finishes the in-flight step, then ends with `JobHaltedOperator`. The Job moves to `HALTED`.

A `JobSimulator` (TimedAction) drips a sample job every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OrchestratorAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains job ledger; reads step ledger. Produces `JobReport` on completion. | `JobWorkflow` | returns typed result to workflow |
| `HttpCallerAgent` | `AutonomousAgent` | Executes HTTP call subtasks from seeded fixtures (`sample-data/http-fixtures.jsonl`). | `JobWorkflow` | — |
| `QueuePublisherAgent` | `AutonomousAgent` | Simulates publishing events to named queues (`sample-data/queue-fixtures.jsonl`). | `JobWorkflow` | — |
| `DbQueryAgent` | `AutonomousAgent` | Returns query results from fixture data (`sample-data/db-fixtures.jsonl`). | `JobWorkflow` | — |
| `ScriptRunnerAgent` | `AutonomousAgent` | Returns simulated script output for an allow-listed script set. | `JobWorkflow` | — |
| `JobWorkflow` | `Workflow` | Drives the plan → dispatch-guarded → execute → sanitize → record → decide loop, plus replan and halt branches. | `JobEndpoint`, `JobRequestConsumer` | `JobEntity` |
| `JobEntity` | `EventSourcedEntity` | Holds the job's lifecycle, job ledger, step ledger, and final report. | `JobWorkflow` | `JobView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `JobEndpoint` (operator action) | `JobWorkflow` (polls) |
| `RequestQueue` | `EventSourcedEntity` | Audit log of submitted jobs. | `JobEndpoint`, `JobSimulator` | `JobRequestConsumer` |
| `JobView` | `View` | List-of-jobs read model for the UI. | `JobEntity` events | `JobEndpoint` |
| `JobRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a `JobWorkflow` per submission. | `RequestQueue` events | `JobWorkflow` |
| `JobSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/job-prompts.jsonl` and enqueues it. | scheduler | `RequestQueue` |
| `StuckJobMonitor` | `TimedAction` | Every 30 s, marks any job stuck in `EXECUTING` past 5 minutes as `STUCK`. | scheduler | `JobEntity` |
| `JobEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, get, list, SSE, operator halt. | — | `JobView`, `RequestQueue`, `JobEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record JobRequest(String prompt, String requestedBy) {}

record JobLedger(
    List<String> facts,
    List<String> missing,
    List<String> plan,
    Optional<DispatchDecision> currentDispatch
) {}

record DispatchDecision(
    ExecutorKind executor,
    String step,
    String rationale
) {}

record StepResult(
    ExecutorKind executor,
    String step,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record StepEntry(
    int attempt,
    ExecutorKind executor,
    String step,
    StepVerdict verdict,
    String scrubbedResult,
    Optional<String> blocker,
    Instant recordedAt
) {}

record StepLedger(List<StepEntry> entries) {}

record JobReport(String summary, List<String> evidence, Instant producedAt) {}

record Job(
    String jobId,
    String prompt,
    JobStatus status,
    Optional<JobLedger> ledger,
    Optional<StepLedger> steps,
    Optional<JobReport> report,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ExecutorKind { HTTP, QUEUE, DB, SCRIPT }
enum StepVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, UNSAFE }
enum JobStatus { PLANNING, EXECUTING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`JobEntity`)

`JobCreated`, `JobPlanned`, `StepDispatched`, `StepBlocked`, `StepRecorded`, `LedgerRevised`, `JobCompleted`, `JobFailed`, `JobHaltedOperator`, `JobFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`RequestQueue`)

`JobSubmitted { jobId, prompt, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ prompt, requestedBy? }` → `202 { jobId }`. Starts a workflow.
- `GET /api/jobs` — list all jobs. Optional `?status=...`.
- `GET /api/jobs/{id}` — one job (full ledgers + report).
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Workflow Integration"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a job, operator halt/resume control, live list of jobs with status pills, expand-row to see the job ledger, the step ledger entries, and the final report.

Browser title: `<title>Akka Sample: Workflow Integration</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `OrchestratorAgent`): every `DispatchDecision` is checked against (a) the executor allow-list, (b) a content policy that forbids HTTP calls to non-allow-listed hosts, queue publishes to system-reserved topics, database mutations, and script execution outside the allow-listed set. Blocking. Failure → `StepBlocked` entry + replan request.
- **HO1 — deployer runtime monitoring** (`hotl`, flavor `deployer-runtime-monitoring`): an operator dashboard pane shows every in-flight job, its current dispatch, and its last step entry. Two buttons — Halt new dispatches, Resume — drive `SystemControlEntity`. The workflow polls `SystemControlEntity` before each dispatch and exits with `JobHaltedOperator` if the flag is set. The pane also surfaces queue-depth and retry counts from the step ledger.

## 9. Agent prompts

- `OrchestratorAgent` → `prompts/orchestrator.md`. Maintains both ledgers; decides next step.
- `HttpCallerAgent` → `prompts/http-caller.md`. Returns HTTP call answers from fixtures.
- `QueuePublisherAgent` → `prompts/queue-publisher.md`. Returns publish confirmations from fixtures.
- `DbQueryAgent` → `prompts/db-query.md`. Returns query results from fixtures.
- `ScriptRunnerAgent` → `prompts/script-runner.md`. Returns simulated script output.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Fetch the latest deployment manifest from the registry, publish a rollout event, query the active instance count, and summarise the state." Job progresses `PLANNING → EXECUTING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE.
2. **J2** — Submit a job whose plan would invoke an HTTP call to a non-allow-listed host. The guardrail blocks the dispatch; the orchestrator replans; the job either completes via a different path or fails after the replan budget is exhausted.
3. **J3** — Submit a job and click **Halt new dispatches** while it is `EXECUTING`. The in-flight step finishes; no further dispatches occur; the job ends in `HALTED`.
4. **J4** — Submit a job whose plan exercises the HTTP caller; one fixture response contains a `Bearer eyJ…`-shaped token. The step ledger entry shows the token replaced by `[REDACTED:bearer-token]`; the Orchestrator's next prompt never contains the literal token.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named event-driven-planner-executor demonstrating the
planner-executor × ops-automation cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
planner-executor-ops-automation-event-driven-planner-executor. Java package
io.akka.samples.workflowintegration. Akka 3.6.0. HTTP port 9124.

Components to wire (exactly):
- 5 AutonomousAgents:
  * OrchestratorAgent — definition() with capabilities:
      capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(DECIDE).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(COMPOSE_REPORT).maxIterationsPerTask(2)).
    System prompt from prompts/orchestrator.md. PLAN returns JobLedger.
    DECIDE returns a NextStep tagged union (DispatchDecision | Replan |
    Complete | Fail). COMPOSE_REPORT returns JobReport.
  * HttpCallerAgent — capability(TaskAcceptance.of(HTTP_CALL).maxIterationsPerTask(2)).
    Prompt from prompts/http-caller.md. Returns StepResult.
  * QueuePublisherAgent — capability(TaskAcceptance.of(QUEUE_PUBLISH).maxIterationsPerTask(2)).
    Prompt from prompts/queue-publisher.md. Returns StepResult.
  * DbQueryAgent — capability(TaskAcceptance.of(DB_QUERY).maxIterationsPerTask(2)).
    Prompt from prompts/db-query.md. Returns StepResult.
  * ScriptRunnerAgent — capability(TaskAcceptance.of(RUN_SCRIPT).maxIterationsPerTask(2)).
    Prompt from prompts/script-runner.md. Returns StepResult.

- 1 Workflow JobWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> guardrailStep ->
  dispatchStep -> sanitizeStep -> recordStep -> decideStep
  -> [back to checkHaltStep or to composeReportStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45),
    dispatchStep ofSeconds(120) (covers any executor call),
    decideStep ofSeconds(45), composeReportStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(JobWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions
  to haltedStep (emits JobHaltedOperator on JobEntity).
  guardrailStep runs the deterministic vetter over DispatchDecision; on
  reject records a StepBlocked entry via JobEntity.recordBlock(step, reason)
  and loops back to proposeStep.
  dispatchStep uses switch on DispatchDecision.executor to call the matching
  executor agent via forAutonomousAgent(...).runSingleTask(...) then
  forJob(jobId).result(...).
  sanitizeStep applies CredentialScrubber.scrub to the StepResult.content.
  recordStep calls JobEntity.recordStep(entry).
  decideStep calls forAutonomousAgent(OrchestratorAgent.class, DECIDE);
  on Continue or Replan loops; on Complete transitions to composeReportStep
  -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity JobEntity holding Job state. emptyState() returns
  Job.initial("", null) with no commandContext() reference. Commands:
  createJob, recordPlan, recordDispatch, recordBlock, recordStep,
  reviseLedger, completeJob, failJob, haltOperator, timeoutFail, getJob.
  Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity RequestQueue with command enqueueJob(jobId, prompt,
  requestedBy) emitting JobSubmitted.

- 1 View JobView with row type JobRow (mirror of Job minus heavy ledger
  payloads — truncate to last 3 step entries plus counts; the UI fetches
  the full job by id on click). ONE query getAllJobs SELECT * AS jobs FROM
  job_view. No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer JobRequestConsumer subscribed to RequestQueue events; on
  JobSubmitted starts a JobWorkflow with jobId as the workflow id.

- 2 TimedActions:
  * JobSimulator — every 90s, reads next line from
    src/main/resources/sample-events/job-prompts.jsonl and calls
    RequestQueue.enqueueJob.
  * StuckJobMonitor — every 30s, queries JobView.getAllJobs, filters
    EXECUTING jobs whose createdAt is older than 5 minutes, calls
    JobEntity.timeoutFail; JobWorkflow polls JobEntity.getJob in its
    decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * JobEndpoint at /api with POST /jobs, GET /jobs (filters client-side
    from getAllJobs), GET /jobs/{id}, GET /jobs/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- OrchestratorTasks.java declaring three Task<R> constants: PLAN
  (resultConformsTo JobLedger), DECIDE (NextStep), COMPOSE_REPORT (JobReport).
- ExecutorTasks.java declaring four Task<R> constants: HTTP_CALL,
  QUEUE_PUBLISH, DB_QUERY, RUN_SCRIPT (all resultConformsTo StepResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface
  with permits Continue, Replan, Complete, Fail (each carrying its own
  payload — Continue with DispatchDecision, Replan with revisedLedger,
  Complete with JobReport stub, Fail with failureReason).
- application/CredentialScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36},
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback for tokens ≥ 32
  chars whose Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token], [REDACTED:jwt],
  [REDACTED:openai-key], [REDACTED:bearer-token], [REDACTED:high-entropy].
- application/StepGuardrail.java — deterministic vetter. Reject if the
  executor is not HTTP/QUEUE/DB/SCRIPT, if an HTTP step names a host not
  on the allow-list (akka.io, doc.akka.io, github.com, registry.example.io),
  if a QUEUE step names a topic prefixed with _sys.*, if a DB step contains
  INSERT/UPDATE/DELETE/DROP keywords, if a SCRIPT step names a script not
  in the allow-listed set (deploy.sh, status.sh, rollback.sh, health.sh).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9124 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/job-prompts.jsonl with 8 canned job
  prompts spanning HTTP, queue, DB, and script needs.
- src/main/resources/sample-data/http-fixtures.jsonl — 10 canned HTTP
  fixture responses (host, path, statusCode, body). ONE fixture body must
  contain a Bearer-token-shaped string for the J4 acceptance test.
- src/main/resources/sample-data/queue-fixtures.jsonl — 8 simulated
  queue publish confirmations (topic, partition, offset, timestamp).
- src/main/resources/sample-data/db-fixtures.jsonl — 8 canned query
  results (queryPattern, rowCount, rows as JSON arrays).
- src/main/resources/sample-data/scripts.jsonl — allow-listed scripts
  with canned output (deploy.sh, status.sh, rollback.sh, health.sh).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, HO1)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations.external_tool_calls, and
  compliance.capabilities; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/orchestrator.md, prompts/http-caller.md,
  prompts/queue-publisher.md, prompts/db-query.md, prompts/script-runner.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Workflow Integration",
  one-line pitch, prerequisites (including the integration form's
  host-software requirement: None), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
  NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs: Overview,
  Architecture (4 mermaid diagrams + click-to-expand component table with
  syntax-highlighted Java snippets), Risk Survey (7 sub-tabs with answers
  from risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix
  (5-column ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows), App UI (form + operator halt/resume control +
  live list with status pills and expand-on-click for ledgers and report).
  Browser title exactly: <title>Akka Sample: Workflow Integration</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var — record the env-var NAME in
        application.conf.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file. No .env, no entry in
  application.conf, no secrets.yaml. Akka records only the REFERENCE.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch on agent class
  name and Task<R> id. Each branch reads JSON from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    orchestrator.json — three lists keyed by task id:
      "PLAN" → 4–6 JobLedger entries (facts, missing, plan steps spanning
      all four executor kinds).
      "DECIDE" → 4–6 NextStep entries covering Continue (all four
      executors), Replan, Complete, Fail.
      "COMPOSE_REPORT" → 4–6 JobReport entries with 60–120 word summaries
      and 3–5 evidence bullets.
    http-caller.json — 6 StepResult entries; ONE body must include the
      literal substring "Bearer eyJhbGciOiJIUzI1NiJ9.example" for J4.
    queue-publisher.json — 5 StepResult entries, ok=true, content is a
      publish confirmation with topic, offset, timestamp.
    db-query.json — 5 StepResult entries with row-count summaries.
    script-runner.json — 5 StepResult entries aligned with the
      allow-listed script set (deploy.sh, status.sh, rollback.sh, health.sh).
- MockModelProvider.seedFor(jobId) makes selection deterministic per job id.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md.
Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent.
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the Job entity state.
- Lesson 7: AutonomousAgent requires companion OrchestratorTasks.java and
  ExecutorTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current
  lineup. Conservative defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: HTTP port 9124 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string "Runs out of the
  box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute.
  No zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build".
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
