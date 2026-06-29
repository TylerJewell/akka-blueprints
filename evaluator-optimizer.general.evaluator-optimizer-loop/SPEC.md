# SPEC — evaluator-optimizer-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Evaluator-Optimizer Loop.
**One-line pitch:** Submit a problem statement; a generator agent produces a candidate solution; an evaluator agent scores it against a quality rubric; the two iterate until the evaluator accepts or the loop hits its attempt ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`GeneratorAgent`) and a reviewer agent (`EvaluatorAgent`), feeding each evaluation back into the next candidate until convergence or a halt. The blueprint also demonstrates two governance mechanisms — an **eval-event** that records every cycle's verdict for downstream quality measurement, and an **output guardrail** that gates each candidate against a deterministic token-count rule before the evaluator runs.

## 3. User-facing flows

The user opens the App UI tab and submits a problem statement (a description plus a token ceiling).

1. The system creates a `Job` record in `GENERATING` and starts an `OptimizationWorkflow`.
2. The Generator produces candidate #1: a concise solution to the problem statement.
3. The output guardrail vets the candidate against the token ceiling. Over-length candidates are short-circuited back to the Generator with a deterministic feedback note; they never reach the Evaluator.
4. The Evaluator scores the candidate against a fixed rubric (accuracy, conciseness, completeness, relevance) and returns either `ACCEPT` with a one-line rationale, or `REVISE` with a typed `EvaluationNotes` payload (three bullets at most).
5. On `ACCEPT`, the workflow transitions the job to `ACCEPTED` with the winning candidate's text and the evaluator's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the evaluation, and the evaluator's verdict on the entity, then calls the Generator again with the evaluation attached. The Generator produces candidate #2.
7. If the loop reaches `maxAttempts` (default 4) without an `ACCEPT`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring candidate is preserved on the entity along with every evaluation for audit, and an `EvalRecorded` event captures the rejection.

A `ProblemSimulator` (TimedAction) drips a canned problem statement every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GeneratorAgent` | `AutonomousAgent` | Produces a candidate solution to a problem statement; accepts prior evaluation feedback on revisions. | `OptimizationWorkflow` | returns `Candidate` to workflow |
| `EvaluatorAgent` | `AutonomousAgent` | Scores a candidate against the rubric; returns `ACCEPT` or `REVISE` with notes. | `OptimizationWorkflow` | returns `Evaluation` to workflow |
| `OptimizationWorkflow` | `Workflow` | Runs the generate → guardrail → evaluate → revise loop; halts at the ceiling. | `OptimizationEndpoint`, `SubmissionConsumer` | `JobEntity` |
| `JobEntity` | `EventSourcedEntity` | Holds the job lifecycle, every attempt, every evaluation, and the final outcome. | `OptimizationWorkflow` | `JobsView` |
| `SubmissionQueue` | `EventSourcedEntity` | Logs each submitted problem statement for replay and audit. | `OptimizationEndpoint`, `ProblemSimulator` | `SubmissionConsumer` |
| `JobsView` | `View` | List-of-jobs read model. | `JobEntity` events | `OptimizationEndpoint` |
| `SubmissionConsumer` | `Consumer` | Subscribes to `SubmissionQueue` events; starts a workflow per submission. | `SubmissionQueue` events | `OptimizationWorkflow` |
| `ProblemSimulator` | `TimedAction` | Drips a sample problem every 60 s from `sample-events/problems.jsonl`. | scheduler | `SubmissionQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `JobsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `JobEntity` |
| `OptimizationEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `JobsView`, `SubmissionQueue`, `JobEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record ProblemStatement(String description, int tokenCeiling, String submittedBy) {}

record Candidate(String text, int tokenCount, Instant generatedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record EvaluationNotes(List<String> bullets, String overallRationale) {}

record Evaluation(EvaluatorVerdict verdict, EvaluationNotes notes, int score, Instant evaluatedAt) {}

record Attempt(
    int attemptNumber,
    Candidate candidate,
    GuardrailVerdict guardrail,
    Optional<Evaluation> evaluation
) {}

record Job(
    String jobId,
    String description,
    int tokenCeiling,
    int maxAttempts,
    JobStatus status,
    List<Attempt> attempts,
    Optional<Integer> acceptedAttemptNumber,
    Optional<String> acceptedText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus { GENERATING, EVALUATING, ACCEPTED, REJECTED_FINAL }

enum EvaluatorVerdict { ACCEPT, REVISE }
```

### Events (on `JobEntity`)

`JobCreated`, `AttemptGenerated`, `AttemptGuardrailVerdictRecorded`, `AttemptEvaluated`, `JobAccepted`, `JobRejectedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ description, tokenCeiling?, submittedBy? }` → `{ jobId }`. Starts a workflow.
- `GET /api/jobs` — list all jobs. Optional `?status=GENERATING|EVALUATING|ACCEPTED|REJECTED_FINAL`.
- `GET /api/jobs/{id}` — one job (including every attempt and every evaluation).
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Evaluator-Optimizer Loop"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a problem statement, live list of jobs with status pills, click-to-expand per-attempt timeline showing each candidate, the guardrail verdict, the evaluator's verdict, and the evaluator's notes.

Browser title: `<title>Akka Sample: Evaluator-Optimizer Loop</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`after-llm-response` on `GeneratorAgent`): a deterministic check that the candidate's token count is at or below the per-job ceiling. Over-length candidates short-circuit back to the Generator with a structured feedback note (`reasonCode = OVER_CEILING`); they never reach the Evaluator. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's evaluation is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, ceilingExceeded }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/jobs/{id}`.

## 9. Agent prompts

- `GeneratorAgent` → `prompts/generator.md`. Produces a candidate solution to a problem statement; on a revision call, takes the prior `EvaluationNotes` as input and produces a revised candidate.
- `EvaluatorAgent` → `prompts/evaluator.md`. Scores a candidate against the fixed rubric; returns `ACCEPT` with a one-line rationale or `REVISE` with three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a problem statement; job progresses `GENERATING` → `EVALUATING` → `ACCEPTED` within the attempt ceiling; the App UI shows every attempt's candidate and evaluation.
2. **J2 — halt at ceiling** — Submit a problem whose rubric is impossible (test mode forces the Evaluator to `REVISE` every attempt); job progresses through every attempt and lands in `REJECTED_FINAL` with the best candidate preserved and a structured rejection reason.
3. **J3 — guardrail block** — Submit a problem with `tokenCeiling = 50`; the Generator's first candidate exceeds the ceiling; the guardrail short-circuits with `reasonCode = OVER_CEILING`, the Generator re-drafts under the ceiling, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any job shows one `EvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named evaluator-optimizer-loop demonstrating the evaluator-optimizer
× general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-evaluator-optimizer-loop.
Java package io.akka.samples.evaluatoroptimizer. Akka 3.6.0. HTTP port 9401.

Components to wire (exactly):
- 2 AutonomousAgents:
  * GeneratorAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_CANDIDATE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/generator.md. Returns Candidate{text,
    tokenCount, generatedAt} for both GENERATE and REVISE_CANDIDATE. The
    REVISE_CANDIDATE task takes (originalProblem, priorCandidate, EvaluationNotes)
    as inputs.
  * EvaluatorAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE).maxIterationsPerTask(2)). System
    prompt from prompts/evaluator.md. Returns Evaluation{verdict, notes, score,
    evaluatedAt} where verdict is the EvaluatorVerdict enum (ACCEPT | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow OptimizationWorkflow with steps:
    startStep -> generateStep -> guardrailStep -> [guardrail FAIL? generateStep
    again with structured feedback : evaluateStep] ->
    [verdict ACCEPT? acceptStep : (attemptCount < maxAttempts ?
       generateStep with evaluation attached : rejectStep)] -> END.
  generateStep calls forAutonomousAgent(GeneratorAgent.class, jobId).runSingleTask(
    GENERATE or REVISE_CANDIDATE) then forTask(taskId).result(GENERATE or
    REVISE_CANDIDATE). evaluateStep calls forAutonomousAgent(EvaluatorAgent.class,
    jobId).runSingleTask(EVALUATE). acceptStep emits JobAccepted.
    rejectStep emits JobRejectedFinal with the highest-scoring attempt's
    text as best-of and a structured rejectionReason. Override settings()
    with stepTimeout(60s) on generateStep and evaluateStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  guardrailStep is a pure-function step (no LLM call): checks
    candidate.tokenCount() <= ceiling. On FAIL, emits
    AttemptGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "OVER_CEILING", then transitions back to generateStep with a
    structured feedback EvaluationNotes("Candidate exceeds the configured
    token ceiling; shorten and resubmit.").

- 1 EventSourcedEntity JobEntity holding state Job{jobId, description,
  tokenCeiling, maxAttempts, JobStatus status, List<Attempt> attempts,
  Optional<Integer> acceptedAttemptNumber, Optional<String> acceptedText,
  Optional<String> rejectionReason, Instant createdAt, Optional<Instant>
  finishedAt}. JobStatus enum: GENERATING, EVALUATING, ACCEPTED,
  REJECTED_FINAL. Events: JobCreated, AttemptGenerated,
  AttemptGuardrailVerdictRecorded, AttemptEvaluated, JobAccepted,
  JobRejectedFinal, EvalRecorded. Commands: createJob, recordCandidate,
  recordGuardrail, recordEvaluation, accept, rejectFinal, recordEval,
  getJob. emptyState() returns Job.initial("", "", 500, 4) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity SubmissionQueue with command submitProblem(description,
  tokenCeiling, submittedBy) emitting ProblemSubmitted{jobId, description,
  tokenCeiling, submittedBy, submittedAt}.

- 1 View JobsView with row type JobRow (mirrors Job; the attempts list
  is preserved as-is — the list is bounded at maxAttempts so size stays
  reasonable). Table updater consumes JobEntity events. ONE query
  getAllJobs SELECT * AS jobs FROM jobs_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer SubmissionConsumer subscribed to SubmissionQueue events; on
  ProblemSubmitted starts an OptimizationWorkflow with the jobId as the
  workflow id.

- 2 TimedActions:
  * ProblemSimulator — every 60s, reads next line from
    src/main/resources/sample-events/problems.jsonl and calls
    SubmissionQueue.submitProblem.
  * EvalSampler — every 30s, queries JobsView.getAllJobs, finds jobs
    with an evaluated attempt that has not yet been recorded as an
    EvalRecorded event, and calls JobEntity.recordEval(attemptNumber,
    verdict, score, ceilingExceeded). Idempotent per (jobId, attemptNumber).

- 2 HttpEndpoints:
  * OptimizationEndpoint at /api with POST /jobs, GET /jobs, GET /jobs/{id},
    GET /jobs/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /jobs body
    is {description, tokenCeiling?, submittedBy?}; missing tokenCeiling
    defaults to 500, missing submittedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- OptimizationTasks.java declaring three Task<R> constants: GENERATE (resultConformsTo
  Candidate), REVISE_CANDIDATE (Candidate), EVALUATE (Evaluation).
- Domain records Candidate, GuardrailVerdict, EvaluationNotes, Evaluation,
  Attempt, Job; enums JobStatus, EvaluatorVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9401 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  evaluator-optimizer.loop.max-attempts = 4 and
  evaluator-optimizer.loop.default-token-ceiling = 500, overridable by
  env var.
- src/main/resources/sample-events/problems.jsonl with 8 canned problem
  lines, each shaped {"description":"...", "tokenCeiling":500}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  after-llm-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = solution-iteration,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/generator.md, prompts/evaluator.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Evaluator-Optimizer Loop",
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
  per-attempt timeline). Browser title exactly:
  <title>Akka Sample: Evaluator-Optimizer Loop</title>.

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
  named in Section 9: generator.json, evaluator.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    generator.json — 6 Candidate entries. Three are well-formed solutions
      between 300 and 490 tokens addressing the problems in problems.jsonl.
      Two are revision candidates that address prior evaluation notes
      (shorter, more precise). One is an intentionally over-ceiling
      candidate (>550 tokens) used to exercise the guardrail in J3.
    evaluator.json — 6 Evaluation entries. Three return verdict=ACCEPT with
      score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and an EvaluationNotes payload of
      three bullets ("accuracy is imprecise in step 2",
      "completeness gaps in the error-handling section",
      "relevance drifts from the stated problem scope").
- A MockModelProvider.seedFor(jobId, attemptNumber) helper makes the
  selection deterministic per (jobId, attemptNumber) so the same job in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. GeneratorAgent
  and EvaluatorAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an OptimizationTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Job row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: OptimizationTasks.java is mandatory; generating GeneratorAgent or
  EvaluatorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9401, declared in application.conf
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
