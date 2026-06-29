# SPEC — agent-benchmark-harness

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Agent Benchmark Harness.
**One-line pitch:** Trigger a benchmark run; a runner agent submits evaluation tasks to a target model; a scorer agent grades each response against a reference rubric; the harness aggregates results into a pass/fail report and can gate a downstream CI pipeline.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern applied to automated quality measurement: a Workflow fans out a fixed task set across `RunnerAgent` (which elicits responses) and `ScorerAgent` (which grades them), then aggregates the per-task scores into a run-level pass/fail decision. The blueprint wires two governance mechanisms — a **periodic accuracy eval** that records a snapshot event on every scoring cycle for long-term drift tracking, and a **CI gate** that refuses a configurable action when the most recent run's pass rate is below threshold. A halt terminates the run if the workflow exhausts its retry budget without collecting all scores, preserving partial results for audit.

## 3. User-facing flows

The user opens the App UI tab and either clicks "Run benchmark now" or waits for the `RunScheduler` to fire.

1. The system creates a `BenchmarkRun` record in `PENDING` and starts a `BenchmarkRunWorkflow`.
2. The workflow loads the canonical task set from `TaskRegistry` (the seed set bundled in `benchmark-tasks.jsonl`).
3. For each task, the workflow calls `RunnerAgent`, which submits the task prompt to the configured target model and returns a `TaskResponse`.
4. The `ScorerAgent` grades the `TaskResponse` against the reference answer using the configured rubric (exact-match, semantic-overlap, or custom). It returns a `ScoredResult` with a numeric score (0–100) and a `TaskVerdict` (`PASS` or `FAIL`).
5. The workflow emits a `TaskScoredEvent` on `RunEntity` for each completed task. `TaskResultConsumer` reacts and updates running counters.
6. After all tasks are scored, the workflow computes the aggregate pass rate. If it is at or above `passingThreshold` (default 0.80), the run transitions to `PASSED`; otherwise to `FAILED`.
7. A `RunSummaryEvent` is emitted with `{ passRate, passCount, failCount, totalTasks, durationMs }`.

The `RunScheduler` (TimedAction) triggers a new run every 24 hours so the harness tracks quality drift without manual intervention.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RunnerAgent` | `AutonomousAgent` | Submits an evaluation task to the target model; returns a raw `TaskResponse`. | `BenchmarkRunWorkflow` | returns `TaskResponse` to workflow |
| `ScorerAgent` | `AutonomousAgent` | Grades a `TaskResponse` against the reference answer and rubric; returns `ScoredResult`. | `BenchmarkRunWorkflow` | returns `ScoredResult` to workflow |
| `BenchmarkRunWorkflow` | `Workflow` | Fans out all tasks; collects scores; aggregates pass rate; halts on budget exhaustion. | `BenchmarkEndpoint`, `RunScheduler` | `RunEntity` |
| `RunEntity` | `EventSourcedEntity` | Holds the run lifecycle, every task's response and score, and the final aggregate. | `BenchmarkRunWorkflow` | `RunsView` |
| `TaskRegistry` | `EventSourcedEntity` | Stores the canonical task set and reference answers. | `BenchmarkEndpoint` | `BenchmarkRunWorkflow` |
| `RunsView` | `View` | List-of-runs read model. | `RunEntity` events | `BenchmarkEndpoint` |
| `TaskResultConsumer` | `Consumer` | Subscribes to `RunEntity` events; updates run-level counters. | `RunEntity` events | `RunEntity` |
| `RunScheduler` | `TimedAction` | Triggers a new benchmark run every 24 h. | scheduler | `BenchmarkEndpoint` → `BenchmarkRunWorkflow` |
| `AccuracySampler` | `TimedAction` | Every 15 min, queries `RunsView` and records an `AccuracySnapshotRecorded` event for any run that completed since the last tick. | scheduler | `RunEntity` |
| `BenchmarkEndpoint` | `HttpEndpoint` | `/api/runs/*` — trigger, get, list, SSE; `/api/tasks/*` — list tasks; `/api/ci-gate` — gate check; `/api/metadata/*`. | — | `RunsView`, `TaskRegistry`, `RunEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record EvalTask(String taskId, String prompt, String referenceAnswer, String category) {}

record TaskResponse(String taskId, String rawOutput, Instant respondedAt) {}

record ScoredResult(
    String taskId,
    TaskVerdict verdict,
    int score,
    String rationale,
    Instant scoredAt
) {}

record TaskAttempt(
    String taskId,
    TaskResponse response,
    ScoredResult scored
) {}

record RunAggregate(
    int totalTasks,
    int passCount,
    int failCount,
    double passRate,
    long durationMs
) {}

record BenchmarkRun(
    String runId,
    String triggeredBy,
    RunStatus status,
    List<TaskAttempt> taskAttempts,
    Optional<RunAggregate> aggregate,
    Optional<String> failureReason,
    Instant startedAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus { PENDING, RUNNING, PASSED, FAILED }

enum TaskVerdict { PASS, FAIL }
```

### Events (on `RunEntity`)

`RunStarted`, `TaskResponseRecorded`, `TaskScoredEvent`, `RunSummaryEvent`, `RunAborted`, `AccuracySnapshotRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/runs` — body `{ triggeredBy?, taskFilter? }` → `{ runId }`. Starts a workflow.
- `GET /api/runs` — list all runs. Optional `?status=PENDING|RUNNING|PASSED|FAILED`.
- `GET /api/runs/{id}` — one run (including every task attempt, score, and aggregate).
- `GET /api/runs/sse` — server-sent events stream of every run change.
- `GET /api/tasks` — list registered tasks from `TaskRegistry`.
- `GET /api/ci-gate` — returns `{ gated: boolean, runId, passRate, threshold }`. Returns `200` with `gated=false` when the most recent completed run passes; returns `200` with `gated=true` when it fails or no completed run exists.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Agent Benchmark Harness"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-periodic = blue, ci-gate = red).
- **App UI** — button to trigger a run, live list of runs with status pills, click-to-expand per-task timeline showing each task's prompt, the raw response, the scorer's verdict, and the score.

Browser title: `<title>Akka Sample: Agent Benchmark Harness</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-periodic** (`continuous-accuracy`): every 15 minutes, `AccuracySampler` queries `RunsView`, finds completed runs that do not yet have a matching `AccuracySnapshotRecorded` event, and records the snapshot with `{ runId, passRate, passCount, failCount, recordedAt }`. The event also fires once per run on terminal transition (RunSummaryEvent). Enforcement: non-blocking. Surfaces in the App UI's per-run timeline.
- **CI1 — ci-gate** (`test-gate`): `GET /api/ci-gate` reads the most recent completed run from `RunsView` and compares its `passRate` to the configured threshold (`benchmark.passing-threshold`, default 0.80). If `passRate < threshold` or no completed run exists, the endpoint returns `gated=true`; a CI pipeline polling this endpoint treats `gated=true` as a blocking failure. Enforcement: build-gate (caller-enforced; the endpoint is the authority).

## 9. Agent prompts

- `RunnerAgent` → `prompts/runner.md`. Submits an evaluation task prompt to the target model and returns the raw response string.
- `ScorerAgent` → `prompts/scorer.md`. Grades a raw response against the reference answer using the configured rubric; returns a `ScoredResult` with a numeric score and pass/fail verdict.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — passing run** — Trigger a benchmark run; the run progresses `PENDING` → `RUNNING` → `PASSED`; every task's response and score appear in the App UI expanded view.
2. **J2 — failing run** — Trigger a run in test mode where all tasks are forced to score below threshold; the run lands in `FAILED` with the aggregate pass rate and per-task breakdown preserved.
3. **J3 — CI gate check** — After a `FAILED` run, call `GET /api/ci-gate`; response contains `gated=true` with the failing pass rate and the threshold.
4. **J4 — accuracy snapshot timeline** — Expand any completed run; the timeline shows one `AccuracySnapshotRecorded` event with `passRate`, `passCount`, and `failCount`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named agent-benchmark-harness demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-agent-benchmark-harness.
Java package io.akka.samples.benchmark. Akka 3.6.0. HTTP port 9501.

Components to wire (exactly):
- 2 AutonomousAgents:
  * RunnerAgent — definition() with
    capability(TaskAcceptance.of(SUBMIT_TASK).maxIterationsPerTask(2)).
    System prompt loaded from prompts/runner.md. Returns TaskResponse{taskId,
    rawOutput, respondedAt}.
  * ScorerAgent — definition() with
    capability(TaskAcceptance.of(SCORE_TASK).maxIterationsPerTask(2)).
    System prompt from prompts/scorer.md. Returns ScoredResult{taskId, verdict,
    score, rationale, scoredAt} where verdict is the TaskVerdict enum (PASS | FAIL)
    and score is an integer 0–100.

- 1 Workflow BenchmarkRunWorkflow with steps:
    initStep -> [for each task in taskList: runStep(taskId) -> scoreStep(taskId)]
    -> aggregateStep -> [passRate >= threshold ? passStep : failStep] -> END.
  runStep calls forAutonomousAgent(RunnerAgent.class, runId+taskId).runSingleTask(
    SUBMIT_TASK) then forTask(taskId).result(SUBMIT_TASK). scoreStep calls
    forAutonomousAgent(ScorerAgent.class, runId+taskId).runSingleTask(SCORE_TASK).
  aggregateStep is a pure-function step: computes passCount, failCount, passRate,
    durationMs from the collected ScoredResults. Emits RunSummaryEvent.
  passStep emits RunSummaryEvent with status=PASSED. failStep emits RunSummaryEvent
    with status=FAILED plus a structured failureReason ("pass rate N% below threshold M%").
  Override settings() with stepTimeout(90s) on runStep and scoreStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(failStep)).

- 1 EventSourcedEntity RunEntity holding state BenchmarkRun{runId, triggeredBy,
  RunStatus status, List<TaskAttempt> taskAttempts, Optional<RunAggregate> aggregate,
  Optional<String> failureReason, Instant startedAt, Optional<Instant> finishedAt}.
  RunStatus enum: PENDING, RUNNING, PASSED, FAILED.
  Events: RunStarted, TaskResponseRecorded, TaskScoredEvent, RunSummaryEvent,
  RunAborted, AccuracySnapshotRecorded.
  Commands: startRun, recordTaskResponse, recordTaskScore, finalizeRun, abortRun,
  recordAccuracySnapshot, getRun. emptyState() returns BenchmarkRun.initial() with
  PENDING status. Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity TaskRegistry with command registerTask(taskId, prompt,
  referenceAnswer, category) emitting TaskRegistered{taskId, prompt, referenceAnswer,
  category, registeredAt}. Bootstrapped at startup from
  src/main/resources/sample-events/benchmark-tasks.jsonl (10 tasks, each shaped
  {"taskId":"t-001", "prompt":"...", "referenceAnswer":"...", "category":"..."}).

- 1 View RunsView with row type RunRow (mirrors BenchmarkRun; taskAttempts list
  is bounded at maxTasksPerRun so size stays reasonable). Table updater consumes
  RunEntity events. ONE query getAllRuns SELECT * AS runs FROM runs_view. No WHERE
  status filter — caller filters client-side because Akka cannot auto-index enum
  columns (Lesson 2).

- 1 Consumer TaskResultConsumer subscribed to RunEntity events; on TaskScoredEvent
  increments the running pass/fail counters on RunEntity (idempotent on taskId per run).

- 2 TimedActions:
  * RunScheduler — every 24h, calls POST /api/runs with triggeredBy="scheduler".
  * AccuracySampler — every 15 min, queries RunsView.getAllRuns, finds completed
    runs (PASSED or FAILED) with a RunSummaryEvent that has not yet been recorded as
    AccuracySnapshotRecorded, and calls RunEntity.recordAccuracySnapshot(runId,
    passRate, passCount, failCount). Idempotent per runId.

- 2 HttpEndpoints:
  * BenchmarkEndpoint at /api with POST /runs, GET /runs, GET /runs/{id},
    GET /runs/sse, GET /tasks, GET /ci-gate, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/. POST /runs body
    is {triggeredBy?, taskFilter?}; missing triggeredBy defaults to "anonymous";
    taskFilter is an optional list of category strings.
    GET /ci-gate reads the most recent completed run from RunsView and returns
    {gated: boolean, runId: String|null, passRate: double|null, threshold: double}.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- BenchmarkTasks.java declaring two Task<R> constants: SUBMIT_TASK (resultConformsTo
  TaskResponse), SCORE_TASK (resultConformsTo ScoredResult).
- Domain records TaskResponse, ScoredResult, TaskAttempt, RunAggregate, BenchmarkRun,
  EvalTask; enums RunStatus, TaskVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9501
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from the
  canonical env vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-run config
  benchmark.passing-threshold = 0.80 and benchmark.max-tasks-per-run = 50,
  overridable by env var.
- src/main/resources/sample-events/benchmark-tasks.jsonl with 10 canned tasks,
  each shaped {"taskId":"t-001","prompt":"...","referenceAnswer":"...","category":"reasoning"}.
  Categories: reasoning (4), factual (3), instruction-following (3).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (E1 eval-periodic
  continuous-accuracy, CI1 ci-gate test-gate) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = agent-quality-measurement,
  decisions.authority_level = report-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/runner.md, prompts/scorer.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Agent Benchmark Harness",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (trigger button + live list with status pills, click-to-expand
  per-task timeline). Browser title exactly:
  <title>Akka Sample: Agent Benchmark Harness</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning
        the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: runner.json, scorer.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    runner.json — 10 TaskResponse entries matching the 10 bundled tasks.
      Each entry has taskId, rawOutput (a plausible answer string 20–200
      characters), and respondedAt. Four entries are correct (matching the
      reference answer), three are partial, three are wrong — giving a
      natural 40% fail rate for testing the FAILED path.
    scorer.json — 10 ScoredResult entries. Four return verdict=PASS with
      score=85-100 and a rationale. Three return verdict=PASS with score=70-80.
      Three return verdict=FAIL with score=20-50 and a rationale noting what
      the response got wrong.
- A MockModelProvider.seedFor(runId, taskId) helper makes the selection
  deterministic per (runId, taskId) so the same run in dev produces the
  same score trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. RunnerAgent
  and ScorerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a BenchmarkTasks companion declaring the two Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(90s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the BenchmarkRun row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: BenchmarkTasks.java is mandatory; generating RunnerAgent or
  ScorerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9501, declared in application.conf
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
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the
  reference does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements; removed panels are deleted from
  the HTML, not hidden with display:none.
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
