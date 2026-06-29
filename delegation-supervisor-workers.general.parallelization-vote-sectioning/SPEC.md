# SPEC — parallelization-vote-sectioning

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Parallelization Workflow (Sectioning + Voting).
**One-line pitch:** Submit a job; a supervisor splits it across independent section workers OR fans it out for diverse voting — all in parallel — then aggregates the results into one output.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern in its broadest general-purpose form: a Workflow fans identical or partitioned work to multiple AutonomousAgents simultaneously, collects all results, and asks a supervisor to aggregate. The blueprint covers two complementary parallelization modes — **sectioning** (the job is split into non-overlapping parts; each worker handles one) and **voting** (every worker sees the full task; the supervisor applies a vote rule to their diverse outputs). Both modes run through the same Akka primitives; the mode is a field on the job record. The blueprint also demonstrates an **output guardrail** that vets the aggregated output before it is surfaced to the caller, and an **eval-event** governance hook that captures each supervisor aggregation decision as a quality signal.

## 3. User-facing flows

The user opens the App UI tab and submits a job via the form, selecting a mode (SECTIONING or VOTING) and entering a prompt.

1. The system creates a `Job` record in `PARTITIONING` and starts a `ParallelWorkflow`.
2. In SECTIONING mode, the Supervisor decomposes the prompt into N `SectionTask` items. In VOTING mode, the Supervisor prepares a shared `VotePrompt` dispatched to all VoteWorkers.
3. The workflow fans out: all workers run concurrently. Each returns a typed result (`SectionResult` or `VoteResult`).
4. The Supervisor receives all results and produces an `AggregatedOutput { answer, mode, workerOutputs, aggregationVerdict }`.
5. An output guardrail vets the aggregated output. If it fails, the job moves to `BLOCKED`. Otherwise the job moves to `AGGREGATED`.
6. If any worker times out after 60 seconds, the workflow short-circuits: the Supervisor aggregates from whichever results arrived; the job enters `PARTIAL`.

A `JobSimulator` (TimedAction) drips a sample job every 60 seconds so the App UI is non-empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ParallelSupervisor` | `AutonomousAgent` | Partitions jobs into sections or prepares the shared voting prompt; then aggregates all worker results. | `ParallelWorkflow` | returns typed result to workflow |
| `SectionWorker` | `AutonomousAgent` | Processes one section of a partitioned job; returns a `SectionResult`. | `ParallelWorkflow` | — |
| `VoteWorker` | `AutonomousAgent` | Independently evaluates the full prompt; returns a `VoteResult`. | `ParallelWorkflow` | — |
| `ParallelWorkflow` | `Workflow` | Fans work out to all workers concurrently, joins results, calls Supervisor for aggregation, runs the output guardrail. | `JobEndpoint`, `JobRequestConsumer` | `JobEntity` |
| `JobEntity` | `EventSourcedEntity` | Holds the job's lifecycle (PARTITIONING → IN_PROGRESS → AGGREGATED / PARTIAL / BLOCKED). | `ParallelWorkflow` | `JobView` |
| `JobQueue` | `EventSourcedEntity` | Logs each submitted job for replay and audit. | `JobEndpoint`, `JobSimulator` | `JobRequestConsumer` |
| `JobView` | `View` | List-of-jobs read model. | `JobEntity` events | `JobEndpoint` |
| `JobRequestConsumer` | `Consumer` | Listens to `JobQueue` events and starts a workflow per submission. | `JobQueue` events | `ParallelWorkflow` |
| `JobSimulator` | `TimedAction` | Drips a sample job every 60 s. | scheduler | `JobQueue` |
| `EvalSampler` | `TimedAction` | Samples one aggregated job every 5 minutes; emits an `EvalScored` event. | scheduler | `JobEntity` |
| `JobEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, get, list, SSE. | — | `JobView`, `JobQueue`, `JobEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
enum ParallelMode { SECTIONING, VOTING }

record JobRequest(String prompt, ParallelMode mode, String submittedBy) {}

record SectionTask(int sectionIndex, int totalSections, String sectionPrompt) {}
record SectionResult(int sectionIndex, String content, Instant processedAt) {}

record VotePrompt(String prompt, int workerIndex, int totalWorkers) {}
record VoteResult(int workerIndex, String answer, double confidence, Instant votedAt) {}

record Partition(List<SectionTask> sections, int workerCount) {}
record VotePlan(VotePrompt sharedPrompt, int workerCount) {}

record AggregatedOutput(
    String answer,
    ParallelMode mode,
    List<SectionResult> sectionOutputs,
    List<VoteResult> voteOutputs,
    String aggregationVerdict,
    Instant aggregatedAt
) {}

record Job(
    String jobId,
    String prompt,
    ParallelMode mode,
    JobStatus status,
    int workerCount,
    Optional<List<SectionResult>> sectionResults,
    Optional<List<VoteResult>> voteResults,
    Optional<AggregatedOutput> aggregated,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus { PARTITIONING, IN_PROGRESS, AGGREGATED, PARTIAL, BLOCKED }
```

### Events (on `JobEntity`)

`JobCreated`, `WorkersDispatched`, `SectionResultReceived`, `VoteResultReceived`, `JobAggregated`, `JobPartial`, `JobBlocked`, `EvalScored`.

### Events (on `JobQueue`)

`JobSubmitted { jobId, prompt, mode, submittedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ prompt, mode }` → `{ jobId }`. Starts a workflow.
- `GET /api/jobs` — list all jobs. Optional `?status=PARTITIONING|IN_PROGRESS|AGGREGATED|PARTIAL|BLOCKED`.
- `GET /api/jobs/{id}` — one job.
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Parallelization Workflow (Sectioning + Voting)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a job (prompt + mode selector), live list of jobs with status pills, expand-row to see section or vote results, the aggregated answer, and the eval score.

Browser title: `<title>Akka Sample: Parallelization Workflow (Sectioning + Voting)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `ParallelSupervisor`): vets the aggregated output for consistency, incoherence across sections, and explicit policy violations. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one aggregated job every 5 minutes and emits an `EvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `ParallelSupervisor` → `prompts/supervisor.md`. Partitions the job into sections or prepares the voting prompt; later aggregates all worker results.
- `SectionWorker` → `prompts/section-worker.md`. Processes one section; returns `SectionResult`.
- `VoteWorker` → `prompts/vote-worker.md`. Independently evaluates the full prompt; returns `VoteResult`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a SECTIONING job; status progresses PARTITIONING → IN_PROGRESS → AGGREGATED within 90 s; sections arrive close together; the aggregated answer stitches them in order.
2. **J2** — Submit a VOTING job; all workers return independently; the supervisor selects or synthesises the winning answer; status reaches AGGREGATED.
3. **J3** — Inject a worker timeout; job enters PARTIAL; remaining sections are aggregated from what arrived.
4. **J4** — Inject a guardrail failure; job enters BLOCKED with a `failureReason`.
5. **J5** — Wait for the eval sampler; an AGGREGATED job acquires an `evalScore` without delivery being affected.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named parallelization-vote-sectioning demonstrating the
delegation-supervisor-workers × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-general-parallelization-vote-sectioning.
Java package io.akka.samples.parallelizationworkflowsectioningvoting.
Akka 3.6.0. HTTP port 9255.

Components to wire (exactly):
- 3 AutonomousAgents:
  * ParallelSupervisor — definition() with
    capability(TaskAcceptance.of(PARTITION).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(AGGREGATE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/supervisor.md. Returns Partition for
    PARTITION (SECTIONING mode) or VotePlan for PARTITION (VOTING mode) and
    AggregatedOutput{answer, mode, sectionOutputs, voteOutputs,
    aggregationVerdict, aggregatedAt} for AGGREGATE.
  * SectionWorker — capability(TaskAcceptance.of(PROCESS_SECTION)
    .maxIterationsPerTask(3)). System prompt from prompts/section-worker.md.
    Returns SectionResult{sectionIndex, content, processedAt}.
  * VoteWorker — capability(TaskAcceptance.of(CAST_VOTE)
    .maxIterationsPerTask(2)). System prompt from prompts/vote-worker.md.
    Returns VoteResult{workerIndex, answer, confidence, votedAt}.

- 1 Workflow ParallelWorkflow with steps:
  partitionStep -> [parallel] workerStep(0..N-1) -> joinStep ->
  aggregateStep -> guardrailStep -> emitStep.
  partitionStep calls forAutonomousAgent(ParallelSupervisor.class, PARTITION).
  In SECTIONING mode, spawn one SectionWorker call per section (PROCESS_SECTION).
  In VOTING mode, spawn one VoteWorker call per worker slot (CAST_VOTE).
  All worker calls run in parallel via CompletionStage allOf / zip; each
  governed by WorkflowSettings.builder().stepTimeout(60s). On any timeout,
  transition to a partialStep that calls aggregateStep with whichever results
  arrived, then ends with JobPartial.
  aggregateStep calls forAutonomousAgent(ParallelSupervisor.class, AGGREGATE)
  with all results; give aggregateStep a 90s stepTimeout. guardrailStep runs
  a deterministic checker plus LLM judge on the aggregated output; on failure,
  end with JobBlocked. WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity JobEntity holding state Job{jobId, prompt,
  ParallelMode mode, JobStatus status, int workerCount,
  Optional<List<SectionResult>> sectionResults,
  Optional<List<VoteResult>> voteResults,
  Optional<AggregatedOutput> aggregated, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale,
  Instant createdAt, Optional<Instant> finishedAt}.
  JobStatus enum: PARTITIONING, IN_PROGRESS, AGGREGATED, PARTIAL, BLOCKED.
  ParallelMode enum: SECTIONING, VOTING.
  Events: JobCreated, WorkersDispatched, SectionResultReceived,
  VoteResultReceived, JobAggregated, JobPartial, JobBlocked, EvalScored.
  Commands: createJob, dispatchWorkers, receiveSectionResult,
  receiveVoteResult, aggregate, markPartial, block, recordEval, getJob.
  emptyState() returns Job.initial("", null, null) with no commandContext()
  reference.

- 1 EventSourcedEntity JobQueue with command enqueueJob(prompt, mode,
  submittedBy) emitting JobSubmitted{jobId, prompt, mode, submittedBy,
  submittedAt}.

- 1 View JobView with row type JobRow (mirrors Job minus heavy nested lists;
  every nullable field is Optional<T>). Table updater consumes JobEntity
  events. ONE query getAllJobs SELECT * AS jobs FROM job_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller
  filters client-side.

- 1 Consumer JobRequestConsumer subscribed to JobQueue events; on
  JobSubmitted starts a ParallelWorkflow with the jobId as the workflow id.

- 2 TimedActions:
  * JobSimulator — every 60s, reads the next line from
    src/main/resources/sample-events/sample-jobs.jsonl and calls
    JobQueue.enqueueJob. Alternates between SECTIONING and VOTING modes.
  * EvalSampler — every 5 minutes, queries JobView.getAllJobs, picks the
    oldest AGGREGATED job without an evalScore, runs a 1-5 rubric judge
    over the aggregated output, then calls JobEntity.recordEval(score,
    rationale).

- 2 HttpEndpoints:
  * JobEndpoint at /api with POST /jobs, GET /jobs, GET /jobs/{id},
    GET /jobs/sse, and the /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ParallelTasks.java declaring four Task<R> constants: PARTITION (Partition
  or VotePlan), PROCESS_SECTION (SectionResult), CAST_VOTE (VoteResult),
  AGGREGATE (AggregatedOutput).
- Domain records JobRequest, SectionTask, SectionResult, VotePrompt,
  VoteResult, Partition, VotePlan, AggregatedOutput.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9255 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-jobs.jsonl with 8 canned job
  lines (mix of SECTIONING and VOTING prompts).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = parallel-task-aggregation,
  decisions.authority_level = recommend-only,
  data.data_classes.pii = false, capabilities.* = false; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/supervisor.md, prompts/section-worker.md, prompts/vote-worker.md
  loaded at agent startup as system prompts.
- README.md at the project root: title
  "Akka Sample: Parallelization Workflow (Sectioning + Voting)", one-line
  pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Five tabs: Overview, Architecture
  (4 mermaid diagrams + click-to-expand component table with
  syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows),
  App UI (form with prompt input + mode selector + live list with status
  pills). Browser title exactly:
  <title>Akka Sample: Parallelization Workflow (Sectioning + Voting)</title>.
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
  section-worker.json, vote-worker.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    supervisor.json — list of Partition objects (4-6 entries, each with
      2-4 SectionTask items and a matching workerCount) and AggregatedOutput
      objects (4-6 entries, each with an 80-150 word answer, empty sectionOutputs
      and voteOutputs lists, aggregationVerdict = "ok").
    section-worker.json — 4-6 SectionResult entries, each with sectionIndex
      0-3 and 50-100 word content.
    vote-worker.json — 4-6 VoteResult entries, each with workerIndex 0-2, a
      concise answer taking a clear position, and confidence between 0.6 and
      0.95.
- A MockModelProvider.seedFor(jobId) helper makes the selection deterministic
  per job id so the same job produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends
  clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit
  stepTimeout (60s per worker, 90s aggregation); default 5s is too short
  for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion ParallelTasks.java
  declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never
  "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9255 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"),
  never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc). Without these, state
  names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never
  NodeList index; delete removed panels from the DOM, do not display:none
  them.
- Parallel workflow steps use CompletionStage allOf / zip, NOT sequential
  calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export
  block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, marketing tone.
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
