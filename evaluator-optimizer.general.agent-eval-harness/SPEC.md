# SPEC — agent-eval-harness

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Agent Eval Harness.
**One-line pitch:** Trigger a test suite; an evaluator agent drives each case through a target agent; a judge agent scores each response; the harness aggregates accuracy, enforces a quality gate, and records the entire trajectory for regression auditing.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow iterates over a test suite, alternating between an executor agent (`EvaluatorAgent`) that invokes the target agent on each case and a reviewer agent (`JudgeAgent`) that scores each response, then aggregates the results and checks them against a configurable accuracy threshold. The blueprint demonstrates two governance mechanisms — a **ci-gate** that halts the run if accuracy falls below the threshold (blocking, system-level) and an **eval-periodic** that snapshots accuracy scores on a recurring schedule for continuous monitoring (non-blocking, observable end-to-end).

## 3. User-facing flows

The user opens the App UI tab and triggers an eval run (selecting a suite and optionally an accuracy threshold override).

1. The system creates an `EvalRun` record in `PENDING` and starts an `EvalRunWorkflow`.
2. The workflow loads the suite from `SuiteRegistry`, producing a list of `TestCase` records.
3. For each case: the `EvaluatorAgent` calls the target agent with the case input and returns a `CaseResponse`; the `JudgeAgent` scores the response against the expected answer and rubric, returning `PASS` or `FAIL` with a `JudgingNotes` payload.
4. Each case result (`CaseEvaluated` event) is written to `EvalRunEntity` before the next case begins.
5. After all cases are evaluated, the workflow computes `passRate = passCount / totalCases`.
6. If `passRate >= passingThreshold`, the workflow emits `RunPassed`; the run transitions to `PASSED` with `passRate` and `passCount` recorded.
7. If `passRate < passingThreshold`, the ci-gate activates: the workflow emits `GateFailed` with a structured failure reason (`passRate, threshold, failCount`), then `RunFailed`; the run transitions to `FAILED`. All case results are preserved for audit.
8. On terminal transition, an `AccuracySnapshot` event is emitted capturing the final `passRate`, `passCount`, `failCount`, and `suiteName`.

A `RunScheduler` (TimedAction) triggers a periodic run on the default suite every 5 minutes so the App UI is never empty.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EvaluatorAgent` | `AutonomousAgent` | Invokes the target agent on each test case input; returns a `CaseResponse` with the raw output. | `EvalRunWorkflow` | returns `CaseResponse` to workflow |
| `JudgeAgent` | `AutonomousAgent` | Scores a response against expected answer and rubric; returns `PASS` or `FAIL` with `JudgingNotes`. | `EvalRunWorkflow` | returns `Judgment` to workflow |
| `EvalRunWorkflow` | `Workflow` | Drives the evaluate → judge loop over all cases; computes accuracy; applies the CI gate; records terminal outcome. | `EvalEndpoint`, `SuiteEventConsumer` | `EvalRunEntity` |
| `EvalRunEntity` | `EventSourcedEntity` | Holds the run lifecycle, every case result, and the final accuracy score. | `EvalRunWorkflow` | `EvalRunsView` |
| `SuiteRegistry` | `EventSourcedEntity` | Stores test suite definitions (name, cases, expected answers, rubric). | `EvalEndpoint` | `SuiteEventConsumer` |
| `EvalRunsView` | `View` | List-of-runs read model. | `EvalRunEntity` events | `EvalEndpoint` |
| `SuiteEventConsumer` | `Consumer` | Subscribes to `SuiteRegistry` events; starts a workflow per suite-triggered run. | `SuiteRegistry` events | `EvalRunWorkflow` |
| `RunScheduler` | `TimedAction` | Every 5 min, reads next suite definition from `sample-events/eval-suites.jsonl` and triggers a run. | scheduler | `SuiteRegistry` |
| `AccuracySampler` | `TimedAction` | Every 60 s, scans `EvalRunsView`, records an `AccuracySnapshot` event for any run that completed since the last tick. | scheduler | `EvalRunEntity` |
| `EvalEndpoint` | `HttpEndpoint` | `/api/runs/*` — trigger, get, list, SSE; plus `/api/suites/*` and `/api/metadata/*`. | — | `EvalRunsView`, `SuiteRegistry`, `EvalRunEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record TestCase(String caseId, String input, String expectedAnswer, String rubricHint) {}

record CaseResponse(String caseId, String rawOutput, Instant respondedAt) {}

record JudgingNotes(List<String> bullets, String overallRationale) {}

record Judgment(
    JudgeVerdict verdict,
    JudgingNotes notes,
    double confidenceScore,
    Instant judgedAt
) {}

record CaseResult(
    String caseId,
    CaseResponse response,
    Judgment judgment
) {}

record EvalRun(
    String runId,
    String suiteName,
    int totalCases,
    double passingThreshold,
    RunStatus status,
    List<CaseResult> results,
    Optional<Double> passRate,
    Optional<Integer> passCount,
    Optional<Integer> failCount,
    Optional<String> failureReason,
    Instant startedAt,
    Optional<Instant> finishedAt
) {}

record Suite(
    String suiteName,
    List<TestCase> cases,
    String rubric,
    double defaultPassingThreshold
) {}

enum RunStatus { PENDING, RUNNING, PASSED, FAILED }

enum JudgeVerdict { PASS, FAIL }
```

### Events (on `EvalRunEntity`)

`RunCreated`, `RunStarted`, `CaseEvaluated`, `GateFailed`, `RunPassed`, `RunFailed`, `AccuracySnapshot`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/runs` — body `{ suiteName, passingThreshold? }` → `{ runId }`. Starts a workflow.
- `GET /api/runs` — list all runs. Optional `?status=PENDING|RUNNING|PASSED|FAILED`.
- `GET /api/runs/{id}` — one run (including every case result and judgment).
- `GET /api/runs/sse` — server-sent events stream of every run change.
- `POST /api/suites` — body `{ suiteName, cases[], rubric, defaultPassingThreshold? }` → `{ suiteName }`. Registers or replaces a suite.
- `GET /api/suites/{name}` — retrieve a suite definition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Agent Eval Harness"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-periodic = blue, ci-gate = red).
- **App UI** — form to trigger a run, live list of runs with status pills, click-to-expand per-case timeline showing each response, the judge's verdict, and the judge's notes.

Browser title: `<title>Akka Sample: Agent Eval Harness</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-periodic** (`on-decision-eval`): after every run reaches a terminal state, an `AccuracySnapshot` event is emitted capturing `passRate`, `passCount`, `failCount`, `suiteName`, and `runId`. The `AccuracySampler` TimedAction is the canonical writer; the workflow itself also emits a snapshot on every terminal transition. Enforcement: non-blocking. The snapshots surface in the App UI's run timeline and in `/api/runs/{id}`.
- **G1 — ci-gate** (`build-gate`, `test-gate`): before the workflow emits `RunPassed`, it computes `passRate` and checks it against `passingThreshold`. If `passRate < passingThreshold`, the workflow emits `GateFailed` with a structured payload (`{ passRate, threshold, failCount }`), then transitions to `FAILED`. No `RunPassed` event is ever emitted for a run that falls below threshold. Enforcement: blocking (system-level).

## 9. Agent prompts

- `EvaluatorAgent` → `prompts/evaluator.md`. Invokes the target agent (modeled as an inner LLM call) with the case input; returns the raw output as a `CaseResponse`.
- `JudgeAgent` → `prompts/judge.md`. Scores a response against the expected answer and rubric; returns `PASS` with a brief rationale or `FAIL` with up to three specific bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — passing run** — Trigger a suite; run progresses `PENDING` → `RUNNING` → `PASSED`; the App UI shows every case result with verdict and judge notes.
2. **J2 — CI gate failure** — Trigger a run whose accuracy falls below threshold; run transitions to `FAILED` with `GateFailed` recorded and a structured `failureReason`.
3. **J3 — partial suite progress** — Verify that each `CaseEvaluated` event is written before the next case begins; the live SSE stream shows incremental progress.
4. **J4 — accuracy snapshot timeline** — The expanded view of any completed run shows one `AccuracySnapshot` event with the final accuracy figures.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named agent-eval-harness demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-agent-eval-harness.
Java package io.akka.samples.evalsframework. Akka 3.6.0. HTTP port 9796.

Components to wire (exactly):
- 2 AutonomousAgents:
  * EvaluatorAgent — definition() with
    capability(TaskAcceptance.of(INVOKE_CASE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/evaluator.md. Returns CaseResponse{caseId,
    rawOutput, respondedAt}. The INVOKE_CASE task takes (caseInput, expectedAnswer,
    rubricHint) as inputs.
  * JudgeAgent — definition() with
    capability(TaskAcceptance.of(JUDGE_RESPONSE).maxIterationsPerTask(2)).
    System prompt from prompts/judge.md. Returns Judgment{verdict, notes,
    confidenceScore, judgedAt} where verdict is the JudgeVerdict enum (PASS | FAIL)
    and confidenceScore is a 0.0–1.0 double.

- 1 Workflow EvalRunWorkflow with steps:
    startStep -> loadSuiteStep -> [for each case: invokeStep -> judgeStep ->
    recordStep] -> aggregateStep -> [passRate >= threshold? passStep : failStep] -> END.
  invokeStep calls forAutonomousAgent(EvaluatorAgent.class, runId).runSingleTask(
    INVOKE_CASE) then forTask(taskId).result(INVOKE_CASE). judgeStep calls
    forAutonomousAgent(JudgeAgent.class, runId).runSingleTask(JUDGE_RESPONSE).
  aggregateStep is a pure-function step (no LLM call): computes passRate =
    passCount / totalCases. passStep emits RunPassed with passRate and passCount.
  failStep emits GateFailed{passRate, threshold, failCount} then RunFailed
    with failureReason = "accuracy gate: passRate=X below threshold=Y (failCount=Z)".
  Override settings() with stepTimeout(60s) on invokeStep and judgeStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(failStep)).

- 1 EventSourcedEntity EvalRunEntity holding state EvalRun{runId, suiteName,
  totalCases, passingThreshold, RunStatus status, List<CaseResult> results,
  Optional<Double> passRate, Optional<Integer> passCount, Optional<Integer> failCount,
  Optional<String> failureReason, Instant startedAt, Optional<Instant> finishedAt}.
  RunStatus enum: PENDING, RUNNING, PASSED, FAILED. Events: RunCreated, RunStarted,
  CaseEvaluated, GateFailed, RunPassed, RunFailed, AccuracySnapshot.
  Commands: createRun, startRun, recordCase, recordGateFailed, passRun, failRun,
  recordAccuracySnapshot, getRun. emptyState() returns EvalRun.initial with no
  commandContext() reference. Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity SuiteRegistry with command registerSuite(suiteName, cases,
  rubric, defaultPassingThreshold) emitting SuiteRegistered{suiteName, cases,
  rubric, defaultPassingThreshold, registeredAt}, and command triggerRun(suiteName,
  passingThreshold) emitting RunTriggered{runId, suiteName, passingThreshold, triggeredAt}.

- 1 View EvalRunsView with row type EvalRunRow (mirrors EvalRun; the results list
  is preserved as-is — bounded at maxCasesPerRun). Table updater consumes
  EvalRunEntity events. ONE query getAllRuns SELECT * AS runs FROM eval_runs_view.
  No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer SuiteEventConsumer subscribed to SuiteRegistry events; on
  RunTriggered starts an EvalRunWorkflow with the runId as the workflow id.

- 2 TimedActions:
  * RunScheduler — every 5 min, reads next line from
    src/main/resources/sample-events/eval-suites.jsonl and calls
    SuiteRegistry.triggerRun with the default suite name.
  * AccuracySampler — every 60 s, queries EvalRunsView.getAllRuns, finds runs
    that have reached a terminal state but do not yet have an AccuracySnapshot
    event, and calls EvalRunEntity.recordAccuracySnapshot(passRate, passCount,
    failCount, suiteName). Idempotent per runId.

- 2 HttpEndpoints:
  * EvalEndpoint at /api with POST /runs, GET /runs, GET /runs/{id},
    GET /runs/sse, POST /suites, GET /suites/{name}, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /runs body is {suiteName,
    passingThreshold?}; missing passingThreshold defaults to the suite's
    defaultPassingThreshold (fallback 0.8).
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- EvalTasks.java declaring two Task<R> constants: INVOKE_CASE (resultConformsTo
  CaseResponse), JUDGE_RESPONSE (Judgment).
- Domain records TestCase, CaseResponse, JudgingNotes, Judgment, CaseResult,
  EvalRun, Suite; enums RunStatus, JudgeVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9796 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  agent-eval-harness.eval.max-cases-per-run = 20 and
  agent-eval-harness.eval.default-passing-threshold = 0.8, overridable by env var.
- src/main/resources/sample-events/eval-suites.jsonl with 6 canned suite trigger
  lines, each shaped {"suiteName":"default","passingThreshold":0.8}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (E1 eval-periodic
  on-decision-eval, G1 ci-gate build-gate/test-gate) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = agent-quality-measurement,
  decisions.authority_level = automated-assessment, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/evaluator.md, prompts/judge.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Agent Eval Harness",
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
  rows); App UI (form + live run list with status pills, click-to-expand
  per-case timeline). Browser title exactly:
  <title>Akka Sample: Agent Eval Harness</title>.

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
        application.conf.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory;
        gone when the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes for THIS blueprint:
    evaluator.json — 6 CaseResponse entries. Four are correct-looking outputs
      that match the expected answer format. Two are plausible-but-wrong outputs
      used to exercise the FAIL path and the CI gate in J2.
    judge.json — 6 Judgment entries. Four return verdict=PASS with
      confidenceScore=0.85 or 0.95 and a one-sentence rationale. Two return
      verdict=FAIL with confidenceScore=0.3 and a JudgingNotes payload of
      two bullets ("output contradicts expected answer", "key fact missing").
- A MockModelProvider.seedFor(runId, caseId) helper makes selection deterministic
  per (runId, caseId) so the same run in dev produces the same trajectory.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. EvaluatorAgent
  and JudgeAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an EvalTasks companion declaring the two Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the EvalRun row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: EvalTasks.java is mandatory; generating EvaluatorAgent or
  JudgeAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9796, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements.
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
