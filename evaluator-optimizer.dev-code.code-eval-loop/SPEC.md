# SPEC — code-eval-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Code Assistant (AlphaCodium-style).
**One-line pitch:** Type a problem statement; a generator agent produces code; a verifier agent runs import checks and sandboxed tests against it; the two iterate until all tests pass or the retry budget is exhausted.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`GeneratorAgent`) and a verifier agent (`VerifierAgent`), feeding each verification failure back into the next code generation until all tests pass or a halt triggers. The blueprint also demonstrates three governance mechanisms — a **sandbox guardrail** that blocks code from executing forbidden syscalls or unsafe imports before the verifier runs, an **eval-event** that records every cycle's test-pass/fail verdict for downstream quality measurement, and a **ci-gate** that enforces a final test pass before a solution is surfaced as `PASSED` to the user.

## 3. User-facing flows

The user opens the App UI tab and submits a problem statement (a description plus an optional target language).

1. The system creates a `Solution` record in `GENERATING` and starts a `SolutionWorkflow`.
2. The Generator produces attempt #1: a code solution for the problem statement.
3. The sandbox guardrail inspects the code for forbidden imports and syscall patterns. Code that violates the deny-list is short-circuited back to the Generator with a structured feedback note; it never reaches the Verifier.
4. The Verifier runs import resolution and test execution in an isolated context, then returns either `PASS` with a one-line rationale, or `FAIL` with a typed `VerificationNotes` payload (up to five diagnostic lines).
5. On `PASS`, the workflow transitions the solution to `PASSED` with the winning attempt's code and the verifier's rationale.
6. On `FAIL`, the workflow records the attempt, the guardrail verdict, the verification result, and the verifier's verdict on the entity, then calls the Generator again with the diagnostics attached. The Generator produces attempt #2.
7. If the loop reaches `maxAttempts` (default 5) without a `PASS`, the halt mechanism activates: the workflow ends with `EXHAUSTED`, the best-scoring attempt is preserved on the entity along with every diagnostic for audit, and an `EvalRecorded` event captures the exhaustion.

A `ProblemSimulator` (TimedAction) drips a canned problem every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GeneratorAgent` | `AutonomousAgent` | Produces a code solution for a problem statement; accepts prior verification diagnostics on revisions. | `SolutionWorkflow` | returns `CodeAttempt` to workflow |
| `VerifierAgent` | `AutonomousAgent` | Runs import checks and sandboxed tests against a code attempt; returns `PASS` or `FAIL` with diagnostics. | `SolutionWorkflow` | returns `Verification` to workflow |
| `SolutionWorkflow` | `Workflow` | Runs the generate → sandbox-check → verify → revise loop; halts at the budget ceiling. | `CodeEndpoint`, `ProblemConsumer` | `SolutionEntity` |
| `SolutionEntity` | `EventSourcedEntity` | Holds the solution lifecycle, every attempt, every verification result, and the final outcome. | `SolutionWorkflow` | `SolutionsView` |
| `ProblemQueue` | `EventSourcedEntity` | Logs each submitted problem for replay and audit. | `CodeEndpoint`, `ProblemSimulator` | `ProblemConsumer` |
| `SolutionsView` | `View` | List-of-solutions read model. | `SolutionEntity` events | `CodeEndpoint` |
| `ProblemConsumer` | `Consumer` | Subscribes to `ProblemQueue` events; starts a workflow per submission. | `ProblemQueue` events | `SolutionWorkflow` |
| `ProblemSimulator` | `TimedAction` | Drips a sample problem every 60 s from `sample-events/coding-problems.jsonl`. | scheduler | `ProblemQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `SolutionsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `SolutionEntity` |
| `CodeEndpoint` | `HttpEndpoint` | `/api/solutions/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `SolutionsView`, `ProblemQueue`, `SolutionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Problem(String statement, String language, String submittedBy) {}

record CodeAttempt(String code, String language, int lineCount, Instant generatedAt) {}

record SandboxVerdict(boolean passed, String reasonCode, String detail) {}

record VerificationNotes(List<String> diagnostics, String overallRationale) {}

record Verification(VerifierVerdict verdict, VerificationNotes notes, int score, Instant verifiedAt) {}

record Attempt(
    int attemptNumber,
    CodeAttempt code,
    SandboxVerdict sandbox,
    Optional<Verification> verification
) {}

record Solution(
    String solutionId,
    String statement,
    String language,
    int maxAttempts,
    SolutionStatus status,
    List<Attempt> attempts,
    Optional<Integer> passedAttemptNumber,
    Optional<String> passedCode,
    Optional<String> exhaustionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SolutionStatus { GENERATING, VERIFYING, PASSED, EXHAUSTED }

enum VerifierVerdict { PASS, FAIL }
```

### Events (on `SolutionEntity`)

`SolutionCreated`, `AttemptGenerated`, `AttemptSandboxVerdictRecorded`, `AttemptVerified`, `SolutionPassed`, `SolutionExhausted`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/solutions` — body `{ statement, language?, submittedBy? }` → `{ solutionId }`. Starts a workflow.
- `GET /api/solutions` — list all solutions. Optional `?status=GENERATING|VERIFYING|PASSED|EXHAUSTED`.
- `GET /api/solutions/{id}` — one solution (including every attempt and every verification).
- `GET /api/solutions/sse` — server-sent events stream of every solution change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Code Assistant (AlphaCodium-style)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red, ci-gate = orange).
- **App UI** — form to submit a problem statement, live list of solutions with status pills, click-to-expand per-attempt timeline showing each code attempt, the sandbox verdict, the verifier's verdict, and the verifier's diagnostics.

Browser title: `<title>Akka Sample: Code Assistant (AlphaCodium-style)</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — sandbox guardrail** (`before-tool-call` on `VerifierAgent`): a deterministic scan of the generated code for forbidden import patterns and syscall markers before any execution. Code that matches the deny-list is short-circuited back to the Generator with a structured feedback note (`reasonCode = FORBIDDEN_IMPORT` or `reasonCode = UNSAFE_SYSCALL`); the Verifier never executes it. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's verification result is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, sandboxBlocked }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/solutions/{id}`.
- **CG1 — ci-gate** (`test-gate`): a final deterministic check that the accepted `PASS` result corresponds to a zero-failure test run before the solution is transitioned to `PASSED` and returned to the caller. This prevents a `VerifierAgent` hallucination from surfacing broken code as passing. Enforcement: build-gate.

## 9. Agent prompts

- `GeneratorAgent` → `prompts/generator.md`. Produces a code solution for the problem statement; on a revision call, takes the prior `VerificationNotes` as input and produces a new attempt.
- `VerifierAgent` → `prompts/verifier.md`. Runs import checks and test execution; returns `PASS` with a one-line rationale or `FAIL` with up to five diagnostic lines.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a problem statement; solution progresses `GENERATING` → `VERIFYING` → `PASSED` within the retry budget; the App UI shows every attempt's code and verification result.
2. **J2 — budget exhaustion** — Submit a problem whose tests are impossible (test mode forces the Verifier to `FAIL` every attempt); solution progresses through every attempt and lands in `EXHAUSTED` with the best attempt preserved and a structured exhaustion reason.
3. **J3 — sandbox block** — Submit a problem that causes the Generator to produce code with a forbidden import; the sandbox short-circuits with `reasonCode = FORBIDDEN_IMPORT`, the Generator re-drafts without the violation, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any solution shows one `EvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named code-eval-loop demonstrating the evaluator-optimizer ×
dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-dev-code-code-eval-loop.
Java package io.akka.samples.codeassistantalphacodiumstyle. Akka 3.6.0. HTTP port 9241.

Components to wire (exactly):
- 2 AutonomousAgents:
  * GeneratorAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_CODE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/generator.md. Returns CodeAttempt{code,
    language, lineCount, generatedAt} for both GENERATE and REVISE_CODE. The
    REVISE_CODE task takes (originalProblem, priorCode, VerificationNotes) as
    inputs.
  * VerifierAgent — definition() with
    capability(TaskAcceptance.of(VERIFY).maxIterationsPerTask(2)). System
    prompt from prompts/verifier.md. Returns Verification{verdict, notes, score,
    verifiedAt} where verdict is the VerifierVerdict enum (PASS | FAIL)
    and score is a 0–100 integer test-pass percentage.

- 1 Workflow SolutionWorkflow with steps:
    startStep -> generateStep -> sandboxStep -> [sandbox FAIL? generateStep
    again with structured feedback : verifyStep] ->
    [verdict PASS? passStep : (attemptCount < maxAttempts ?
       generateStep with diagnostics attached : exhaustStep)] -> END.
  generateStep calls forAutonomousAgent(GeneratorAgent.class, solutionId).runSingleTask(
    GENERATE or REVISE_CODE) then forTask(taskId).result(GENERATE or
    REVISE_CODE). verifyStep calls forAutonomousAgent(VerifierAgent.class,
    solutionId).runSingleTask(VERIFY). passStep emits SolutionPassed.
    exhaustStep emits SolutionExhausted with the highest-scoring attempt's
    code as best-of and a structured exhaustionReason. Override settings()
    with stepTimeout(90s) on generateStep and verifyStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep)).
  sandboxStep is a pure-function step (no LLM call): scans
    attempt.code.code() for patterns in the forbidden-import deny-list and
    unsafe-syscall markers. On FAIL, emits
    AttemptSandboxVerdictRecorded with verdict.passed = false and
    reasonCode = "FORBIDDEN_IMPORT" or "UNSAFE_SYSCALL", then transitions
    back to generateStep with a structured feedback VerificationNotes
    listing the offending pattern.

- 1 EventSourcedEntity SolutionEntity holding state Solution{solutionId,
  statement, language, maxAttempts, SolutionStatus status, List<Attempt>
  attempts, Optional<Integer> passedAttemptNumber, Optional<String> passedCode,
  Optional<String> exhaustionReason, Instant createdAt, Optional<Instant>
  finishedAt}. SolutionStatus enum: GENERATING, VERIFYING, PASSED, EXHAUSTED.
  Events: SolutionCreated, AttemptGenerated, AttemptSandboxVerdictRecorded,
  AttemptVerified, SolutionPassed, SolutionExhausted, EvalRecorded. Commands:
  createSolution, recordAttempt, recordSandbox, recordVerification, pass,
  exhaust, recordEval, getSolution. emptyState() returns Solution.initial
  with no commandContext() reference. Event-applier wraps lifecycle fields
  with Optional.of(...).

- 1 EventSourcedEntity ProblemQueue with command enqueueProblem(statement,
  language, submittedBy) emitting ProblemSubmitted{solutionId, statement,
  language, submittedBy, submittedAt}.

- 1 View SolutionsView with row type SolutionRow (mirrors Solution; the
  attempts list is preserved as-is — the list is bounded at maxAttempts so
  size stays reasonable). Table updater consumes SolutionEntity events. ONE
  query getAllSolutions SELECT * AS solutions FROM solutions_view. No WHERE
  status filter — caller filters client-side because Akka cannot auto-index
  enum columns (Lesson 2).

- 1 Consumer ProblemConsumer subscribed to ProblemQueue events; on
  ProblemSubmitted starts a SolutionWorkflow with the solutionId as the
  workflow id.

- 2 TimedActions:
  * ProblemSimulator — every 60s, reads next line from
    src/main/resources/sample-events/coding-problems.jsonl and calls
    ProblemQueue.enqueueProblem.
  * EvalSampler — every 30s, queries SolutionsView.getAllSolutions, finds
    solutions with a verified attempt that has not yet been recorded as an
    EvalRecorded event, and calls SolutionEntity.recordEval(attemptNumber,
    verdict, score, sandboxBlocked). Idempotent per (solutionId, attemptNumber).

- 2 HttpEndpoints:
  * CodeEndpoint at /api with POST /solutions, GET /solutions, GET
    /solutions/{id}, GET /solutions/sse, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/. The POST
    /solutions body is {statement, language?, submittedBy?}; missing language
    defaults to "java", missing submittedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- CodeTasks.java declaring three Task<R> constants: GENERATE (resultConformsTo
  CodeAttempt), REVISE_CODE (CodeAttempt), VERIFY (Verification).
- Domain records CodeAttempt, SandboxVerdict, VerificationNotes, Verification,
  Attempt, Solution; enums SolutionStatus, VerifierVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9241 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  code-eval-loop.solution.max-attempts = 5 and
  code-eval-loop.solution.default-language = "java", overridable by
  env var.
- src/main/resources/sample-events/coding-problems.jsonl with 8 canned
  problem lines, each shaped {"statement":"...", "language":"java"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 sandbox guardrail
  before-tool-call, E1 eval-event on-decision-eval, CG1 ci-gate test-gate)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = code-generation-iteration,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.content-generation = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/generator.md, prompts/verifier.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Code Assistant
  (AlphaCodium-style)", one-line pitch, prerequisites, generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
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
  <title>Akka Sample: Code Assistant (AlphaCodium-style)</title>.

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
  named in Section 9: generator.json, verifier.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    generator.json — 6 CodeAttempt entries. Three are valid Java solutions
      for problems in coding-problems.jsonl (20–40 lines, compilable
      structure). Two are revision attempts that address prior diagnostics
      (shorter methods, renamed variables). One contains a forbidden import
      (e.g. "import java.lang.Runtime") used to exercise the sandbox in J3.
    verifier.json — 6 Verification entries. Three return verdict=PASS with
      score=100 and a one-sentence rationale. Three return verdict=FAIL
      with score=40 or 60 and a VerificationNotes payload of up to five
      diagnostics ("NullPointerException at line 12", "missing null check
      on input", "test testEmptyInput failed: expected 0 got -1").
- A MockModelProvider.seedFor(solutionId, attemptNumber) helper makes the
  selection deterministic per (solutionId, attemptNumber) so the same
  solution in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. GeneratorAgent
  and VerifierAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a CodeTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(90s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Solution row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: CodeTasks.java is mandatory; generating GeneratorAgent or
  VerifierAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9241, declared in application.conf
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
