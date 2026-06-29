# SPEC — tau2-benchmark-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Tau2BenchmarkAgent.
**One-line pitch:** A user submits a tau2 benchmark task definition; one AI agent executes the task's steps (passed as a task attachment, never as inline prompt text) and returns a structured `TaskResult` — PASS / FAIL / PARTIAL with per-step output and a latency measurement.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general benchmark domain. One `BenchmarkAgent` (AutonomousAgent) carries the entire execution; the surrounding components only prepare its input and audit its output. One governance mechanism is wired around the agent:

- An **on-decision periodic evaluator** runs immediately after each `ResultRecorded` event, scoring the agent's task result for output completeness, step coverage, and answer correctness against the task's reference answer. The evaluator is rule-based — no LLM call — which preserves the single-agent promise and ensures that the same result always scores the same.

The blueprint shows that benchmark infrastructure itself benefits from the same governance discipline applied to production agents: structured outputs, deterministic scoring, and audit trails that outlive the session.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **task category** from a dropdown (web-navigation, tool-use, multi-step-reasoning) or pastes a custom task definition JSON.
2. The user clicks **Run task**. The UI POSTs to `/api/runs` and receives a `runId`.
3. The run card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `EXECUTING`.
4. Within ~10–60 s, the workflow's `executeStep` completes. The card transitions to `RESULT_RECORDED`. The result appears: a top-level outcome badge (PASS / FAIL / PARTIAL), total steps attempted, each step's output, and a latency measurement.
5. Within ~1 s of the result, the `scoreStep` finishes. The card shows a **performance score** chip (1–5) plus a one-line rationale describing whether the result's evidence of task completion is solid.
6. The live aggregate panel at the top updates: pass rate, mean score, per-category breakdown.
7. The user can submit another task; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BenchmarkEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `BenchmarkRunEntity`, `BenchmarkView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BenchmarkRunEntity` | `EventSourcedEntity` | Per-run lifecycle: submitted → executing → result → scored. Source of truth. | `BenchmarkEndpoint`, `BenchmarkRunWorkflow` | `BenchmarkView` |
| `BenchmarkRunWorkflow` | `Workflow` | One workflow per run. Steps: `executeStep` → `scoreStep`. | started by `BenchmarkEndpoint` after entity creation | `BenchmarkAgent`, `BenchmarkRunEntity`, `TaskScorer` |
| `BenchmarkAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the task definition as a task attachment; returns `TaskResult`. | invoked by `BenchmarkRunWorkflow` | returns result |
| `TaskScorer` | supporting class | Deterministic rule-based scorer. No LLM call. Inputs: `TaskResult` + `BenchmarkTask`. Outputs: `PerformanceScore`. | invoked by `BenchmarkRunWorkflow` inside `scoreStep` | returns score |
| `BenchmarkView` | `View` | Read model: one row per run for the UI plus an aggregate summary row. | `BenchmarkRunEntity` events | `BenchmarkEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record BenchmarkTask(
    String taskId,
    String category,          // "web-navigation" | "tool-use" | "multi-step-reasoning"
    String description,
    List<String> steps,       // ordered step descriptions
    String referenceAnswer,   // expected final answer for scoring
    int maxSteps
) {}

record RunRequest(
    String runId,
    BenchmarkTask task,
    String submittedBy,
    Instant submittedAt
) {}

record StepOutput(
    int stepIndex,
    String output,
    boolean attempted
) {}

record TaskResult(
    Outcome outcome,
    List<StepOutput> stepOutputs,
    String finalAnswer,
    long latencyMs,
    Instant completedAt
) {}
enum Outcome { PASS, FAIL, PARTIAL }

record PerformanceScore(
    int score,                // 1..5
    String rationale,
    Instant scoredAt
) {}

record BenchmarkRun(
    String runId,
    Optional<RunRequest> request,
    Optional<TaskResult> result,
    Optional<PerformanceScore> score,
    RunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus {
    SUBMITTED, EXECUTING, RESULT_RECORDED, SCORED, FAILED
}
```

Events on `BenchmarkRunEntity`: `RunSubmitted`, `ExecutionStarted`, `ResultRecorded`, `RunScored`, `RunFailed`.

Every nullable lifecycle field on the `BenchmarkRun` state record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ task: BenchmarkTask, submittedBy }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/runs/aggregate` — aggregate performance stats (pass rate, mean score, per-category counts).
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Tau2 Benchmark Agent</title>`.

The App UI tab is a two-column layout: a left rail with the aggregate panel (pass rate gauge, mean score, per-category bar) and the live list of run cards (status pill + outcome badge + score chip + task id + age), and a right pane with the selected run's detail — task description, steps list with outputs, final answer, latency, outcome badge, and score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-decision eval** (`eval-periodic`, `performance-monitor`): runs immediately after `ResultRecorded` lands, as `scoreStep` inside the workflow. A deterministic rule-based `TaskScorer` (no LLM call — the eval is rule-based on purpose, so the same result always scores the same) checks: (1) every task step was attempted (`stepOutputs` length equals `task.steps` length), (2) each `StepOutput.output` is non-empty, (3) `finalAnswer` is non-empty and matches the `referenceAnswer` by exact-match or normalized-string comparison, (4) `latencyMs` is within a reasonable bound (>0 and ≤ 120 000 ms). Scoring rubric: all four criteria met = 5; three met = 4; two met = 3; one met = 2; none met = 1. Emits `RunScored` with a 1–5 score and a one-line rationale. Runs with score ≤ 2 are flagged in the UI for human inspection.

## 9. Agent prompts

- `BenchmarkAgent` → `prompts/benchmark-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached task definition, execute each step in sequence, record its output, and return one `TaskResult` with a `finalAnswer`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded web-navigation task; within 60 s the result appears with one `StepOutput` per task step and a performance score chip.
2. **J2** — A run whose `TaskResult` has all empty `stepOutputs` and an empty `finalAnswer` receives a score of 1 with a clear rationale; the UI flags the card with a red border.
3. **J3** — A run whose `finalAnswer` exactly matches the `referenceAnswer` and all steps are non-empty receives a score of 5; the aggregate pass-rate gauge increments.
4. **J4** — A task that the agent cannot complete within `executeStep`'s 60 s timeout transitions the run to `FAILED`; the partial entity state is preserved and visible in the UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named tau2benchmarkagent demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-tau2-benchmark. Java package io.akka.samples.tau2benchmarkagent.
Akka 3.6.0. HTTP port 9330.

Components to wire (exactly):

- 1 AutonomousAgent BenchmarkAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/benchmark-agent.md>) and
  .capability(TaskAcceptance.of(BenchmarkTasks.EXECUTE_TASK).maxIterationsPerTask(3)).
  The task receives the task description as its instruction text and the full BenchmarkTask
  JSON as a task ATTACHMENT named "task.json" (NOT as inline prompt text — Akka's
  TaskDef.attachment(name, contentBytes) is the canonical call). Output: TaskResult{outcome:
  Outcome (PASS/FAIL/PARTIAL), stepOutputs: List<StepOutput>, finalAnswer: String,
  latencyMs: long, completedAt: Instant}. No guardrail is wired on this agent — the
  single control is the post-result on-decision eval.

- 1 Workflow BenchmarkRunWorkflow per runId with two steps:
  * executeStep — emits ExecutionStarted, then calls
    componentClient.forAutonomousAgent(BenchmarkAgent.class, "agent-" + runId)
      .runSingleTask(
        TaskDef.instructions(run.request.task.description())
          .attachment("task.json", serializeTask(run.request.task).getBytes())
      ) — returns a taskId, then forTask(taskId).result(BenchmarkTasks.EXECUTE_TASK)
    to fetch the result. On success calls BenchmarkRunEntity.recordResult(result).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(BenchmarkRunWorkflow::error).
  * scoreStep — runs a deterministic rule-based TaskScorer (NOT an LLM call) over the
    recorded result: checks that stepOutputs.size() == task.steps().size(), each
    StepOutput.output is non-empty, finalAnswer is non-empty and equals
    task.referenceAnswer() (case-insensitive, trimmed), and latencyMs is in (0, 120000].
    Scoring: 4 criteria met = 5; 3 = 4; 2 = 3; 1 = 2; 0 = 1. Emits RunScored{score:
    1-5, rationale: String}. WorkflowSettings.stepTimeout 5s. error step transitions
    the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity BenchmarkRunEntity (one per runId). State BenchmarkRun{runId:
  String, request: Optional<RunRequest>, result: Optional<TaskResult>, score:
  Optional<PerformanceScore>, status: RunStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. RunStatus enum: SUBMITTED, EXECUTING, RESULT_RECORDED,
  SCORED, FAILED. Events: RunSubmitted{request}, ExecutionStarted{}, ResultRecorded{result},
  RunScored{score}, RunFailed{reason}. Commands: submit, markExecuting, recordResult,
  recordScore, fail, getRun. emptyState() returns BenchmarkRun.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 View BenchmarkView with row type RunRow (mirrors BenchmarkRun). Table updater
  consumes BenchmarkRunEntity events. TWO queries: getAllRuns (SELECT * AS runs FROM
  benchmark_view) and no WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side. The aggregate stats (pass rate, mean score,
  per-category counts) are computed client-side in the UI from the full list.

- 2 HttpEndpoints:
  * BenchmarkEndpoint at /api with POST /runs (body {task: {taskId, category,
    description, steps: [String], referenceAnswer, maxSteps}, submittedBy}; mints runId;
    calls BenchmarkRunEntity.submit; starts BenchmarkRunWorkflow; returns {runId}),
    GET /runs (list from getAllRuns, sorted newest-first), GET /runs/{id} (one row),
    GET /runs/sse (Server-Sent Events forwarded from the view's stream-updates),
    GET /runs/aggregate (computed on the full list — pass rate, mean score,
    per-category pass/fail counts), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- BenchmarkTasks.java declaring one Task<R> constant: EXECUTE_TASK = Task.name("Execute
  benchmark task").description("Execute the given benchmark task steps and produce a
  TaskResult").resultConformsTo(TaskResult.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records BenchmarkTask, RunRequest, StepOutput, TaskResult, Outcome,
  PerformanceScore, BenchmarkRun, RunStatus.

- TaskScorer.java — pure deterministic logic (no LLM). Inputs: TaskResult and BenchmarkTask.
  Outputs: PerformanceScore. Scoring rubric documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9330 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The BenchmarkAgent.definition() binds
  the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/tasks.jsonl with 3 seeded task definitions:
  a 4-step web-navigation task, a 5-step tool-use task, and a 6-step multi-step-reasoning
  task. Each has a non-trivial referenceAnswer so TaskScorer has real work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (E1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false,
  decisions.authority_level = automated (the agent's output is the benchmark result —
  no human acts on it in real time), oversight.human_in_loop = false (benchmark
  infrastructure; humans review aggregates, not individual runs), failure.failure_modes
  including "partial-execution", "wrong-final-answer", "step-skip", "latency-timeout";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/benchmark-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Tau2 Benchmark Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  aggregate panel + live list of run cards; right = selected-run detail with task steps,
  step outputs, final answer, latency, outcome badge, and score chip).
  Browser title exactly: <title>Akka Sample: Tau2 Benchmark Agent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(runId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    execute-benchmark-task.json — 9 TaskResult entries covering the three Outcome values.
      Each entry has a stepOutputs array whose size matches the seeded task's step count,
      a non-empty finalAnswer, and a positive latencyMs. Outcome distribution: 4 PASS,
      3 PARTIAL, 2 FAIL. Plus 1 evidence-thin entry (all stepOutputs have empty output
      strings and an empty finalAnswer) — TaskScorer scores this entry 1, exercising J2.
      The mock selects the evidence-thin entry on every 4th run (modulo seed) so J2 is
      reproducible.
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BenchmarkAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion BenchmarkTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (executeStep
  60s, scoreStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the BenchmarkRun state record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: BenchmarkTasks.java with EXECUTE_TASK = Task.name(...).description(...)
  .resultConformsTo(TaskResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9330 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (BenchmarkAgent). The
  on-decision eval is rule-based (TaskScorer.java) and does NOT make an LLM call — keeping
  the pattern's "one agent" promise honest.
- The task definition is passed as a task ATTACHMENT named "task.json", never inlined into
  the agent's instructions. Verify the generated executeStep uses TaskDef.attachment(...)
  and not string interpolation into the instruction text.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
