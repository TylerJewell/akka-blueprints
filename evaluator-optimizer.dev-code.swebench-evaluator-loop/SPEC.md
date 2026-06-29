# SPEC — swebench-evaluator-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** SWE-Bench Coding Agent.
**One-line pitch:** Submit a GitHub issue and a file context; a patch agent generates a candidate fix; a test-evaluator agent scores it against the test suite; the two iterate until the tests pass or the loop hits its budget ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`PatchAgent`) and a reviewer agent (`TestEvaluatorAgent`), feeding each test failure back into the next patch attempt until all tests pass or a halt activates. The blueprint also demonstrates three governance mechanisms — a **ci-gate** that validates the structural correctness of every patch before the test evaluator runs, an **eval-event** that records every iteration's test outcome for downstream quality measurement, and a **halt** that ends the loop at a token/iteration budget ceiling without leaving the issue in a degenerate state.

## 3. User-facing flows

The user opens the App UI tab and submits a GitHub issue (repo name, issue number, a description, and relevant file context).

1. The system creates an `Issue` record in `PATCHING` and starts a `SolvingWorkflow`.
2. The PatchAgent produces patch attempt #1: a unified diff against the provided file context.
3. The CI gate validates the diff structurally (well-formed hunks, no binary content, within line-count bounds). Malformed patches are short-circuited back to the PatchAgent with a structured feedback note; they never reach the TestEvaluatorAgent.
4. The TestEvaluatorAgent runs the patched test suite (or scores against a ground-truth oracle in the sandbox) and returns either `PASS` with a passing test count, or `FAIL` with a `TestFailureSummary` payload (failing test names, error snippets).
5. On `PASS`, the workflow transitions the issue to `SOLVED` with the winning patch and a summary of tests passed.
6. On `FAIL`, the workflow records the attempt, the gate verdict, the test result, and the failure summary on the entity, then calls the PatchAgent again with the failure details attached. The PatchAgent produces patch attempt #2.
7. If the loop reaches `maxIterations` (default 5) without a `PASS`, the halt mechanism activates: the workflow ends with `EXHAUSTED`, the closest-passing attempt is preserved on the entity along with every test result for audit, and an `EvalRecorded` event captures the exhaustion.

An `IssueSimulator` (TimedAction) drips a canned issue every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PatchAgent` | `AutonomousAgent` | Generates a unified diff against provided file context; accepts prior test failure on revisions. | `SolvingWorkflow` | returns `PatchAttempt` to workflow |
| `TestEvaluatorAgent` | `AutonomousAgent` | Scores a patch by running the test suite; returns `PASS` or `FAIL` with structured failure details. | `SolvingWorkflow` | returns `TestResult` to workflow |
| `SolvingWorkflow` | `Workflow` | Runs the patch → ci-gate → evaluate → revise loop; halts at the ceiling. | `IssueEndpoint`, `IssueIngestConsumer` | `IssueEntity` |
| `IssueEntity` | `EventSourcedEntity` | Holds the issue lifecycle, every patch attempt, every test result, and the final outcome. | `SolvingWorkflow` | `IssuesView` |
| `IssueQueue` | `EventSourcedEntity` | Logs each submitted issue for replay and audit. | `IssueEndpoint`, `IssueSimulator` | `IssueIngestConsumer` |
| `IssuesView` | `View` | List-of-issues read model. | `IssueEntity` events | `IssueEndpoint` |
| `IssueIngestConsumer` | `Consumer` | Subscribes to `IssueQueue` events; starts a workflow per submission. | `IssueQueue` events | `SolvingWorkflow` |
| `IssueSimulator` | `TimedAction` | Drips a sample issue every 90 s from `sample-events/swebench-issues.jsonl`. | scheduler | `IssueQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `IssuesView`, records an `EvalRecorded` event for any iteration that completed since the last tick. | scheduler | `IssueEntity` |
| `IssueEndpoint` | `HttpEndpoint` | `/api/issues/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `IssuesView`, `IssueQueue`, `IssueEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record IssueContext(String repoName, int issueNumber, String description, String fileContext) {}

record PatchAttempt(String unifiedDiff, int lineCount, Instant patchedAt) {}

record CiGateVerdict(boolean passed, String reasonCode, String detail) {}

record TestFailureSummary(List<String> failingTests, List<String> errorSnippets, String overallSummary) {}

record TestResult(
    EvalVerdict verdict,
    TestFailureSummary failures,
    int passCount,
    int failCount,
    Instant evaluatedAt
) {}

record Iteration(
    int iterationNumber,
    PatchAttempt patch,
    CiGateVerdict gate,
    Optional<TestResult> testResult
) {}

record Issue(
    String issueId,
    String repoName,
    int issueNumber,
    String description,
    int maxIterations,
    int tokenBudget,
    IssueStatus status,
    List<Iteration> iterations,
    Optional<Integer> solvedAtIteration,
    Optional<String> acceptedPatch,
    Optional<String> exhaustionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum IssueStatus { PATCHING, EVALUATING, SOLVED, EXHAUSTED }

enum EvalVerdict { PASS, FAIL }
```

### Events (on `IssueEntity`)

`IssueCreated`, `IterationPatched`, `IterationGateVerdictRecorded`, `IterationEvaluated`, `IssueSolved`, `IssueExhausted`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/issues` — body `{ repoName, issueNumber, description, fileContext?, maxIterations? }` → `{ issueId }`. Starts a workflow.
- `GET /api/issues` — list all issues. Optional `?status=PATCHING|EVALUATING|SOLVED|EXHAUSTED`.
- `GET /api/issues/{id}` — one issue (including every iteration and every test result).
- `GET /api/issues/sse` — server-sent events stream of every issue change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "SWE-Bench Coding Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, ci-gate = red, halt = red).
- **App UI** — form to submit an issue, live list of issues with status pills, click-to-expand per-iteration timeline showing each patch, the CI gate verdict, the test evaluator's verdict, and the failure summary.

Browser title: `<title>Akka Sample: SWE-Bench Coding Agent</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **C1 — ci-gate** (`test-gate`): a deterministic structural check on the unified diff before the TestEvaluatorAgent runs. Malformed patches (unparseable hunks, binary content markers, line count exceeding the configured ceiling) are short-circuited back to the PatchAgent with a structured feedback note (`reasonCode = MALFORMED_DIFF` or `EXCEEDS_LINE_BUDGET`); they never enter the test runner. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every iteration's test result is recorded as an `EvalRecorded` event with `{ iterationNumber, verdict, passCount, failCount, gateBlocked }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-iteration timeline and in `/api/issues/{id}`.
- **HT1 — halt** (`automatic-safety-halt`): when the loop reaches `maxIterations` without a `PASS`, or when cumulative token usage crosses `tokenBudget`, the workflow ends with `IssueExhausted`. The entity preserves every patch, every test result, the closest-passing iteration's diff, and a structured exhaustion reason. The system never deletes patches or terminates abruptly. Enforcement: system-level.

## 9. Agent prompts

- `PatchAgent` → `prompts/patch-agent.md`. Generates a unified diff fixing the issue described; on a revision call, takes the prior `TestFailureSummary` as input and produces a revised patch.
- `TestEvaluatorAgent` → `prompts/test-evaluator-agent.md`. Scores a patch against the test suite oracle; returns `PASS` with a passing test count or `FAIL` with a structured `TestFailureSummary`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit an issue; issue progresses `PATCHING` → `EVALUATING` → `SOLVED` within the iteration ceiling; the App UI shows every iteration's patch and test result.
2. **J2 — halt at ceiling** — Submit an issue whose test constraints are impossible (test mode forces the TestEvaluatorAgent to `FAIL` every attempt); issue progresses through every iteration and lands in `EXHAUSTED` with the closest-passing patch preserved and a structured exhaustion reason.
3. **J3 — ci-gate block** — Submit an issue where the PatchAgent's first output is a malformed diff; the gate short-circuits with `reasonCode = MALFORMED_DIFF`, the PatchAgent re-patches, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any issue shows one `EvalRecorded` event per evaluated iteration and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named swebench-evaluator-loop demonstrating the evaluator-optimizer ×
dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-dev-code-swebench-evaluator-loop.
Java package io.akka.samples.swebenchcodingagent. Akka 3.6.0. HTTP port 9890.

Components to wire (exactly):
- 2 AutonomousAgents:
  * PatchAgent — definition() with
    capability(TaskAcceptance.of(GENERATE_PATCH).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_PATCH).maxIterationsPerTask(3)).
    System prompt loaded from prompts/patch-agent.md. Returns PatchAttempt{unifiedDiff,
    lineCount, patchedAt} for both GENERATE_PATCH and REVISE_PATCH. The
    REVISE_PATCH task takes (issueContext, priorPatch, TestFailureSummary) as
    inputs.
  * TestEvaluatorAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE_PATCH).maxIterationsPerTask(2)). System
    prompt from prompts/test-evaluator-agent.md. Returns TestResult{verdict,
    failures, passCount, failCount, evaluatedAt} where verdict is the EvalVerdict
    enum (PASS | FAIL).

- 1 Workflow SolvingWorkflow with steps:
    startStep -> patchStep -> gateStep -> [gate FAIL? patchStep
    again with structured feedback : evaluateStep] ->
    [verdict PASS? solveStep : (iterationCount < maxIterations AND
       tokenUsage < tokenBudget ? patchStep with failure summary attached :
       exhaustStep)] -> END.
  patchStep calls forAutonomousAgent(PatchAgent.class, issueId).runSingleTask(
    GENERATE_PATCH or REVISE_PATCH) then forTask(taskId).result(GENERATE_PATCH or
    REVISE_PATCH). evaluateStep calls forAutonomousAgent(TestEvaluatorAgent.class,
    issueId).runSingleTask(EVALUATE_PATCH). solveStep emits IssueSolved.
  exhaustStep emits IssueExhausted with the highest-passCount iteration's
    diff as best-of and a structured exhaustionReason ("max iterations reached"
    or "token budget exhausted" plus counts). Override settings()
    with stepTimeout(120s) on patchStep and evaluateStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep)).
  gateStep is a pure-function step (no LLM call): validates the unified diff is
    parseable (starts with --- / +++ hunks, no binary markers, lineCount <=
    configured line budget). On FAIL, emits IterationGateVerdictRecorded with
    verdict.passed = false and reasonCode = "MALFORMED_DIFF" or
    "EXCEEDS_LINE_BUDGET", then transitions back to patchStep with a structured
    feedback TestFailureSummary(failingTests=[], errorSnippets=[detail],
    overallSummary="Patch did not pass structural validation; see detail.").

- 1 EventSourcedEntity IssueEntity holding state Issue{issueId, repoName,
  issueNumber, description, maxIterations, tokenBudget, IssueStatus status,
  List<Iteration> iterations, Optional<Integer> solvedAtIteration,
  Optional<String> acceptedPatch, Optional<String> exhaustionReason,
  Instant createdAt, Optional<Instant> finishedAt}. IssueStatus enum:
  PATCHING, EVALUATING, SOLVED, EXHAUSTED. Events: IssueCreated,
  IterationPatched, IterationGateVerdictRecorded, IterationEvaluated,
  IssueSolved, IssueExhausted, EvalRecorded. Commands: createIssue,
  recordPatch, recordGate, recordTestResult, solve, exhaust, recordEval,
  getIssue. emptyState() returns Issue.initial("", "", 0, "", 5, 50000) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity IssueQueue with command enqueueIssue(repoName,
  issueNumber, description, fileContext, maxIterations, tokenBudget) emitting
  IssueSubmitted{issueId, repoName, issueNumber, description, fileContext,
  submittedAt}.

- 1 View IssuesView with row type IssueRow (mirrors Issue; the iterations list
  is preserved as-is — the list is bounded at maxIterations so size stays
  reasonable). Table updater consumes IssueEntity events. ONE query
  getAllIssues SELECT * AS issues FROM issues_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer IssueIngestConsumer subscribed to IssueQueue events; on
  IssueSubmitted starts a SolvingWorkflow with the issueId as the
  workflow id.

- 2 TimedActions:
  * IssueSimulator — every 90s, reads next line from
    src/main/resources/sample-events/swebench-issues.jsonl and calls
    IssueQueue.enqueueIssue.
  * EvalSampler — every 30s, queries IssuesView.getAllIssues, finds issues
    with an evaluated iteration that has not yet been recorded as an
    EvalRecorded event, and calls IssueEntity.recordEval(iterationNumber,
    verdict, passCount, failCount, gateBlocked). Idempotent per
    (issueId, iterationNumber).

- 2 HttpEndpoints:
  * IssueEndpoint at /api with POST /issues, GET /issues, GET /issues/{id},
    GET /issues/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /issues body
    is {repoName, issueNumber, description, fileContext?, maxIterations?,
    tokenBudget?}; missing maxIterations defaults to 5, missing tokenBudget
    defaults to 50000, missing fileContext defaults to "".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- CodingTasks.java declaring three Task<R> constants: GENERATE_PATCH (resultConformsTo
  PatchAttempt), REVISE_PATCH (PatchAttempt), EVALUATE_PATCH (TestResult).
- Domain records PatchAttempt, CiGateVerdict, TestFailureSummary, TestResult,
  Iteration, Issue, IssueContext; enums IssueStatus, EvalVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9890 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  swebench.solving.max-iterations = 5,
  swebench.solving.token-budget = 50000,
  swebench.solving.max-diff-lines = 500, overridable by env var.
- src/main/resources/sample-events/swebench-issues.jsonl with 8 canned issue
  lines, each shaped {"repoName":"...","issueNumber":N,"description":"..."}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (C1 ci-gate test-gate,
  E1 eval-event on-decision-eval, HT1 halt automatic-safety-halt) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = code-patch-generation,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/patch-agent.md, prompts/test-evaluator-agent.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: SWE-Bench Coding Agent",
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
  rows); App UI (form + live list with status pills, click-to-expand
  per-iteration timeline). Browser title exactly:
  <title>Akka Sample: SWE-Bench Coding Agent</title>.

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
  named in Section 9: patch-agent.json, test-evaluator-agent.json), picks
  one entry pseudo-randomly per call, and deserialises it into the agent's
  typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    patch-agent.json — 6 PatchAttempt entries. Three are syntactically
      valid unified diffs within 200 lines that address common Python bug
      patterns (off-by-one, missing return, wrong comparator). Two are
      "revision" patches that tighten an earlier attempt by fixing the
      remaining failing test's assertion. One is an intentionally malformed
      diff (missing --- / +++ header) used to exercise the ci-gate in J3.
    test-evaluator-agent.json — 6 TestResult entries. Three return
      verdict=PASS with passCount=8-12 and failCount=0. Three return
      verdict=FAIL with passCount=5-7, failCount=2-4, and a
      TestFailureSummary with two failing test names and one error snippet.
- A MockModelProvider.seedFor(issueId, iterationNumber) helper makes the
  selection deterministic per (issueId, iterationNumber) so the same issue
  in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. PatchAgent
  and TestEvaluatorAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a CodingTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(120s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Issue row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: CodingTasks.java is mandatory; generating PatchAgent or
  TestEvaluatorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9890, declared in application.conf
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
