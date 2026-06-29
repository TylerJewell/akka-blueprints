# SPEC — competitive-coding-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** USACO Competitive Programming Agent.
**One-line pitch:** Submit a USACO-style problem; a solver agent generates a complete solution; a judge agent runs the code against all test cases in a resource-bounded sandbox and scores the result; the two iterate until every test case passes or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`SolverAgent`) and a reviewer agent (`JudgeAgent`), feeding each structured failure report back into the next solution until convergence or a halt. The blueprint also demonstrates three governance mechanisms — a **sandbox guardrail** that gates each generated solution against resource limits before the judge scores it (control G1), a **test-gate** that treats an all-cases-pass result as the exit condition and a CI-gate trigger (control CG1), and an **eval-event** that records every cycle's verdict for continuous quality tracking (control E1). The sandbox interaction uses E2B (configurable to Modal or Robocorp) as the execution backend.

## 3. User-facing flows

The user opens the App UI tab and submits a USACO-style problem statement (title, problem text, sample input/output pairs, and an optional time-limit override).

1. The system creates a `Submission` record in `GENERATING` and starts a `SolvingWorkflow`.
2. The Solver produces attempt #1: a complete Java program that reads from stdin and writes to stdout, shaped to USACO's IO conventions.
3. The sandbox guardrail compiles and executes the solution inside a resource-bounded E2B micro-VM against all provided test cases. If the sandbox run exceeds the time limit or memory ceiling, the guardrail blocks the attempt and returns structured feedback to the Solver; the solution never reaches the Judge.
4. The Judge analyzes the sandbox execution report (per-case verdicts, runtime, memory peak) and returns either `PASS` — all cases accepted — or `FAIL` with a typed `FailureNotes` payload (three bullets at most: algorithm correctness, edge cases missed, complexity analysis).
5. On `PASS`, the workflow transitions the submission to `ACCEPTED` with the winning solution and the Judge's rationale.
6. On `FAIL`, the workflow records the attempt, the sandbox report, the judge verdict, and the failure notes on the entity, then calls the Solver again with the failure notes attached. The Solver produces attempt #2.
7. If the loop reaches `maxAttempts` (default 5) without a `PASS`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring attempt is preserved on the entity along with every judge report for audit, and an `EvalRecorded` event captures the rejection outcome.

A `ProblemSimulator` (TimedAction) drips a canned USACO-style problem every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SolverAgent` | `AutonomousAgent` | Generates a complete Java solution for a USACO-style problem; accepts prior failure notes on revision calls. | `SolvingWorkflow` | returns `GeneratedSolution` to workflow |
| `JudgeAgent` | `AutonomousAgent` | Analyzes a sandbox execution report; returns `PASS` or `FAIL` with structured notes. | `SolvingWorkflow` | returns `JudgeVerdict` to workflow |
| `SolvingWorkflow` | `Workflow` | Runs the generate → sandbox-gate → judge → revise loop; halts at the ceiling. | `SolverEndpoint`, `ProblemConsumer` | `SubmissionEntity` |
| `SubmissionEntity` | `EventSourcedEntity` | Holds the submission lifecycle, every solution, every sandbox report, every judge verdict, and the final outcome. | `SolvingWorkflow` | `SubmissionsView` |
| `ProblemQueue` | `EventSourcedEntity` | Logs each submitted problem for replay and audit. | `SolverEndpoint`, `ProblemSimulator` | `ProblemConsumer` |
| `SubmissionsView` | `View` | List-of-submissions read model. | `SubmissionEntity` events | `SolverEndpoint` |
| `ProblemConsumer` | `Consumer` | Subscribes to `ProblemQueue` events; starts a workflow per submission. | `ProblemQueue` events | `SolvingWorkflow` |
| `ProblemSimulator` | `TimedAction` | Drips a sample USACO problem every 90 s from `sample-events/usaco-problems.jsonl`. | scheduler | `ProblemQueue` |
| `EvalSampler` | `TimedAction` | Every 45 s, scans `SubmissionsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `SubmissionEntity` |
| `SolverEndpoint` | `HttpEndpoint` | `/api/submissions/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `SubmissionsView`, `ProblemQueue`, `SubmissionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Problem(
    String title,
    String statementText,
    List<TestCase> sampleCases,
    int timeLimitMs,
    int memoryLimitMb,
    String submittedBy
) {}

record TestCase(String inputData, String expectedOutput) {}

record GeneratedSolution(
    String sourceCode,
    String language,
    int lineCount,
    Instant generatedAt
) {}

record SandboxReport(
    boolean resourcesOk,
    String reasonCode,
    String detail,
    List<CaseResult> caseResults,
    int peakMemoryMb,
    long wallTimeMs
) {}

record CaseResult(
    int caseIndex,
    String actualOutput,
    boolean passed,
    String verdict
) {}

record FailureNotes(List<String> bullets, String overallRationale) {}

record JudgeVerdict(
    JudgeOutcome outcome,
    FailureNotes notes,
    int passedCases,
    int totalCases,
    Instant evaluatedAt
) {}

record Attempt(
    int attemptNumber,
    GeneratedSolution solution,
    SandboxReport sandbox,
    Optional<JudgeVerdict> verdict
) {}

record Submission(
    String submissionId,
    String problemTitle,
    String statementText,
    int timeLimitMs,
    int memoryLimitMb,
    int maxAttempts,
    SubmissionStatus status,
    List<Attempt> attempts,
    Optional<Integer> acceptedAttemptNumber,
    Optional<String> acceptedSourceCode,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SubmissionStatus { GENERATING, JUDGING, ACCEPTED, REJECTED_FINAL }

enum JudgeOutcome { PASS, FAIL }
```

### Events (on `SubmissionEntity`)

`SubmissionCreated`, `AttemptGenerated`, `AttemptSandboxReportRecorded`, `AttemptJudged`, `SubmissionAccepted`, `SubmissionRejectedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/submissions` — body `{ title, statementText, sampleCases?, timeLimitMs?, memoryLimitMb?, submittedBy? }` → `{ submissionId }`. Starts a workflow.
- `GET /api/submissions` — list all submissions. Optional `?status=GENERATING|JUDGING|ACCEPTED|REJECTED_FINAL`.
- `GET /api/submissions/{id}` — one submission (including every attempt and every judge verdict).
- `GET /api/submissions/sse` — server-sent events stream of every submission change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "USACO Competitive Programming Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red, ci-gate = orange).
- **App UI** — form to submit a problem, live list of submissions with status pills, click-to-expand per-attempt timeline showing each solution, the sandbox verdict, the judge's verdict, and the judge's failure notes.

Browser title: `<title>Akka Sample: USACO Competitive Programming Agent</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — sandbox guardrail** (`before-tool-call` on `SolvingWorkflow`): a resource check that executes the generated solution in a bounded E2B micro-VM. Any solution that exceeds the configured time limit or memory ceiling is blocked and returns structured feedback (`reasonCode = TLE` or `MLE`) to the Solver before the Judge is invoked. Enforcement: blocking.
- **CG1 — test-gate** (`test-gate`): the Judge's `PASS` verdict requires every sample test case to produce matching output. A partial pass (some cases correct, some wrong) is treated as `FAIL`. The workflow exits to `ACCEPTED` only when the Judge returns `PASS`. The gate is effectively build-gate-style within the loop — it is what terminates the loop on success. Enforcement: build-gate.
- **E1 — eval-event** (`on-decision-eval`): every cycle's judge outcome is recorded as an `EvalRecorded` event with `{ attemptNumber, outcome, passedCases, totalCases, sandboxOk }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. Events surface in the App UI's per-attempt timeline and in `/api/submissions/{id}`.

## 9. Agent prompts

- `SolverAgent` → `prompts/solver.md`. Generates a complete Java solution for a USACO-style competitive programming problem; on a revision call, takes the prior `FailureNotes` as input and produces a corrected solution.
- `JudgeAgent` → `prompts/judge.md`. Analyzes a sandbox execution report; returns `PASS` with a one-line rationale or `FAIL` with three short bullets describing the algorithm defect, edge cases missed, and complexity concern.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a problem; submission progresses `GENERATING` → `JUDGING` → `ACCEPTED` within the retry ceiling; the App UI shows every attempt's solution and judge verdict.
2. **J2 — halt at ceiling** — Submit a problem whose test cases the solver cannot pass (test mode forces the Judge to `FAIL` every attempt); submission progresses through every attempt and lands in `REJECTED_FINAL` with the best solution preserved and a structured rejection reason.
3. **J3 — sandbox block** — Submit a problem where the generated code enters an infinite loop; the sandbox detects the TLE condition, blocks the attempt with `reasonCode = TLE`, and the Solver re-generates with the TLE detail attached.
4. **J4 — eval-event timeline** — The expanded view of any submission shows one `EvalRecorded` event per judged attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named competitive-coding-agent demonstrating the evaluator-optimizer ×
dev-code cell. Requires E2B_API_KEY (or a configurable sandbox backend) for sandbox
execution; all other components run locally.
Maven group io.akka.samples. Artifact id evaluator-optimizer-dev-code-competitive-coding-agent.
Java package io.akka.samples.usacocompetitiveprogrammingagent. Akka 3.6.0. HTTP port 9346.

Components to wire (exactly):
- 2 AutonomousAgents:
  * SolverAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(4))
    AND capability(TaskAcceptance.of(REVISE_SOLUTION).maxIterationsPerTask(4)).
    System prompt loaded from prompts/solver.md. Returns GeneratedSolution{sourceCode,
    language, lineCount, generatedAt} for both GENERATE and REVISE_SOLUTION. The
    REVISE_SOLUTION task takes (problem, priorSolution, FailureNotes) as inputs.
  * JudgeAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE_SANDBOX).maxIterationsPerTask(2)). System
    prompt from prompts/judge.md. Returns JudgeVerdict{outcome, notes, passedCases,
    totalCases, evaluatedAt} where outcome is the JudgeOutcome enum (PASS | FAIL)
    and passedCases/totalCases are integers.

- 1 Workflow SolvingWorkflow with steps:
    startStep -> generateStep -> sandboxStep -> [sandbox FAIL? generateStep
    again with structured feedback : judgeStep] ->
    [outcome PASS? acceptStep : (attemptCount < maxAttempts ?
       generateStep with failure notes attached : rejectStep)] -> END.
  generateStep calls forAutonomousAgent(SolverAgent.class, submissionId).runSingleTask(
    GENERATE or REVISE_SOLUTION) then forTask(taskId).result(GENERATE or
    REVISE_SOLUTION). judgeStep calls forAutonomousAgent(JudgeAgent.class,
    submissionId).runSingleTask(EVALUATE_SANDBOX). acceptStep emits SubmissionAccepted.
    rejectStep emits SubmissionRejectedFinal with the highest-pass-count attempt's
    source as best-of and a structured rejectionReason. Override settings()
    with stepTimeout(120s) on generateStep and sandboxStep, stepTimeout(60s) on
    judgeStep, and defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  sandboxStep calls the E2B sandbox client (HTTP call from within the step) with the
    generated source code, compiles it, runs it against all sampleCases, enforces
    timeLimitMs and memoryLimitMb, and returns a SandboxReport. If SandboxReport
    .resourcesOk = false (TLE or MLE), emits AttemptSandboxReportRecorded with
    reasonCode set and transitions back to generateStep with structured
    FailureNotes("Solution exceeded resource limits; see reasonCode and detail.").

- 1 EventSourcedEntity SubmissionEntity holding state Submission{submissionId,
  problemTitle, statementText, timeLimitMs, memoryLimitMb, maxAttempts,
  SubmissionStatus status, List<Attempt> attempts, Optional<Integer>
  acceptedAttemptNumber, Optional<String> acceptedSourceCode, Optional<String>
  rejectionReason, Instant createdAt, Optional<Instant> finishedAt}.
  SubmissionStatus enum: GENERATING, JUDGING, ACCEPTED, REJECTED_FINAL.
  Events: SubmissionCreated, AttemptGenerated, AttemptSandboxReportRecorded,
  AttemptJudged, SubmissionAccepted, SubmissionRejectedFinal, EvalRecorded.
  Commands: createSubmission, recordSolution, recordSandbox, recordJudgeVerdict,
  accept, rejectFinal, recordEval, getSubmission. emptyState() returns
  Submission.initial("", "", 2000, 256, 5) with no commandContext() reference.
  Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity ProblemQueue with command enqueueProblem(title,
  statementText, sampleCases, timeLimitMs, memoryLimitMb, submittedBy) emitting
  ProblemSubmitted{submissionId, title, statementText, sampleCases, timeLimitMs,
  memoryLimitMb, submittedBy, submittedAt}.

- 1 View SubmissionsView with row type SubmissionRow (mirrors Submission; the attempts
  list is preserved as-is — bounded at maxAttempts). Table updater consumes
  SubmissionEntity events. ONE query getAllSubmissions SELECT * AS submissions FROM
  submissions_view. No WHERE status filter — caller filters client-side because Akka
  cannot auto-index enum columns (Lesson 2).

- 1 Consumer ProblemConsumer subscribed to ProblemQueue events; on ProblemSubmitted
  starts a SolvingWorkflow with the submissionId as the workflow id.

- 2 TimedActions:
  * ProblemSimulator — every 90s, reads next line from
    src/main/resources/sample-events/usaco-problems.jsonl and calls
    ProblemQueue.enqueueProblem.
  * EvalSampler — every 45s, queries SubmissionsView.getAllSubmissions, finds
    submissions with a judged attempt that has not yet been recorded as an
    EvalRecorded event, and calls SubmissionEntity.recordEval(attemptNumber,
    outcome, passedCases, totalCases, sandboxOk). Idempotent per (submissionId,
    attemptNumber).

- 2 HttpEndpoints:
  * SolverEndpoint at /api with POST /submissions, GET /submissions,
    GET /submissions/{id}, GET /submissions/sse, and three /api/metadata/*
    endpoints serving the YAML/MD files from src/main/resources/metadata/.
    The POST /submissions body is {title, statementText, sampleCases?,
    timeLimitMs?, memoryLimitMb?, submittedBy?}; missing timeLimitMs defaults
    to 2000, missing memoryLimitMb defaults to 256, missing submittedBy defaults
    to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- SolverTasks.java declaring three Task<R> constants: GENERATE (resultConformsTo
  GeneratedSolution), REVISE_SOLUTION (GeneratedSolution), EVALUATE_SANDBOX
  (JudgeVerdict).
- Domain records GeneratedSolution, SandboxReport, CaseResult, FailureNotes,
  JudgeVerdict, Attempt, Submission, Problem, TestCase; enums SubmissionStatus,
  JudgeOutcome.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9346 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  competitive-coding-agent.solving.max-attempts = 5,
  competitive-coding-agent.solving.default-time-limit-ms = 2000,
  competitive-coding-agent.solving.default-memory-limit-mb = 256,
  competitive-coding-agent.sandbox.e2b-api-key = ${?E2B_API_KEY},
  all overridable by env var.
- src/main/resources/sample-events/usaco-problems.jsonl with 6 canned USACO-style
  problems, each shaped {"title":"...","statementText":"...","sampleCases":[...],"timeLimitMs":2000}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 sandbox guardrail
  before-tool-call, CG1 test-gate build-gate, E1 eval-event on-decision-eval)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = competitive-code-generation,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/solver.md, prompts/judge.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: USACO Competitive Programming
  Agent", one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section.
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
  per-attempt timeline). Browser title exactly:
  <title>Akka Sample: USACO Competitive Programming Agent</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json. Per-agent shapes:
    solver.json — 5 GeneratedSolution entries. Three are correct Java programs
      for the problems in usaco-problems.jsonl (simple I/O, prefix sums,
      greedy). One is an intentionally TLE solution (nested O(n^2) loop) used
      to exercise the sandbox guardrail in J3. One is a partially-correct
      solution that passes some cases and fails others.
    judge.json — 5 JudgeVerdict entries. Two return outcome=PASS with
      passedCases=totalCases and a one-sentence rationale. Three return
      outcome=FAIL with passedCases < totalCases and a FailureNotes payload
      of three bullets ("algorithm is O(n^2); needs O(n log n) sorting",
      "edge case: empty input not handled", "integer overflow on large inputs").
- A MockModelProvider.seedFor(submissionId, attemptNumber) helper makes the
  selection deterministic per (submissionId, attemptNumber).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. SolverAgent
  and JudgeAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a SolverTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent or the sandbox has an
  explicit stepTimeout override (120s for generateStep and sandboxStep, 60s
  for judgeStep); the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Submission record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: SolverTasks.java is mandatory.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: the run command is /akka:build.
- Lesson 10: HTTP port is 9346, declared in application.conf dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal.
- Lesson 12: the App UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration tier is shown as "E2B sandbox required" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND
  theme variables for state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. The DOM contains exactly five <section class="tab-panel">
  elements; removed panels are deleted from the HTML, not hidden with display:none.
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
