# SPEC — sql-gen-validator-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** SQL Generator.
**One-line pitch:** Type a natural-language question; a generator agent produces a SQL query; a validator agent checks it against the target schema using `EXPLAIN`; the two iterate until the query is valid or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`GeneratorAgent`) and a reviewer agent (`ValidatorAgent`), feeding each validation failure back into the next generation attempt until the query passes or a halt fires. The blueprint also demonstrates two governance mechanisms: a **guardrail** that blocks mutating SQL (`DROP`, `DELETE`, `UPDATE`, `INSERT`) before the validator ever sees the query, and an **eval-event** that records every cycle's validation outcome for downstream quality measurement.

## 3. User-facing flows

The user opens the App UI tab and submits a natural-language question plus a target schema name.

1. The system creates a `QueryRequest` record in `GENERATING` and starts a `SqlWorkflow`.
2. The GeneratorAgent translates the question into a SQL `SELECT` query for the named schema.
3. The mutation guardrail scans the generated SQL for forbidden keywords (`DROP`, `DELETE`, `UPDATE`, `INSERT`, `TRUNCATE`, `ALTER`). A query containing any of these is short-circuited back to the GeneratorAgent with a deterministic feedback note; it never reaches the ValidatorAgent.
4. The ValidatorAgent runs a schema-aware `EXPLAIN` check: it resolves every table and column reference against the known schema definition, verifies join conditions, and returns either `VALID` with a one-line rationale, or `INVALID` with a typed `ValidationNotes` payload (up to three bullets).
5. On `VALID`, the workflow transitions the request to `ACCEPTED` with the passing query and the validator's rationale.
6. On `INVALID`, the workflow records the attempt, the guardrail verdict, the validation result, and the validator's notes on the entity, then calls the GeneratorAgent again with the notes attached. The GeneratorAgent produces the next attempt.
7. If the loop reaches `maxAttempts` (default 4) without a `VALID`, the halt fires: the workflow ends with `FAILED_FINAL`, the best-scoring attempt is preserved on the entity along with every attempt and every validation note for audit, and a `ValidationEvalRecorded` event captures the rejection.

A `RequestSimulator` (TimedAction) drips a canned natural-language question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GeneratorAgent` | `AutonomousAgent` | Translates a natural-language question into SQL; accepts prior validation notes on revisions. | `SqlWorkflow` | returns `GeneratedQuery` to workflow |
| `ValidatorAgent` | `AutonomousAgent` | Runs EXPLAIN-style schema check; returns `VALID` or `INVALID` with notes. | `SqlWorkflow` | returns `ValidationResult` to workflow |
| `SqlWorkflow` | `Workflow` | Drives the generate → guardrail → validate → revise loop; halts at the ceiling. | `SqlEndpoint`, `SubmissionConsumer` | `QueryRequestEntity` |
| `QueryRequestEntity` | `EventSourcedEntity` | Holds the request lifecycle, every attempt's SQL, every validation result, and the final outcome. | `SqlWorkflow` | `QueryRequestView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted question for replay and audit. | `SqlEndpoint`, `RequestSimulator` | `SubmissionConsumer` |
| `QueryRequestView` | `View` | List-of-requests read model. | `QueryRequestEntity` events | `SqlEndpoint` |
| `SubmissionConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a workflow per submission. | `RequestQueue` events | `SqlWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample question every 60 s from `sample-events/sql-questions.jsonl`. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `QueryRequestView`, records a `ValidationEvalRecorded` event for any cycle that completed since the last tick. | scheduler | `QueryRequestEntity` |
| `SqlEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `QueryRequestView`, `RequestQueue`, `QueryRequestEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record NlQuestion(String text, String schemaName, String requestedBy) {}

record GeneratedQuery(String sql, String dialect, Instant generatedAt) {}

record MutationGuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record ValidationNotes(List<String> bullets, String overallRationale) {}

record ValidationResult(
    ValidatorVerdict verdict,
    ValidationNotes notes,
    int score,
    Instant evaluatedAt
) {}

record QueryAttempt(
    int attemptNumber,
    GeneratedQuery query,
    MutationGuardrailVerdict guardrail,
    Optional<ValidationResult> validation
) {}

record QueryRequest(
    String requestId,
    String question,
    String schemaName,
    int maxAttempts,
    QueryStatus status,
    List<QueryAttempt> attempts,
    Optional<Integer> acceptedAttemptNumber,
    Optional<String> acceptedSql,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus { GENERATING, VALIDATING, ACCEPTED, FAILED_FINAL }

enum ValidatorVerdict { VALID, INVALID }
```

### Events (on `QueryRequestEntity`)

`QueryRequestCreated`, `AttemptGenerated`, `AttemptMutationGuardrailVerdictRecorded`, `AttemptValidated`, `QueryRequestAccepted`, `QueryRequestFailedFinal`, `ValidationEvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/queries` — body `{ question, schemaName?, requestedBy? }` → `{ requestId }`. Starts a workflow.
- `GET /api/queries` — list all requests. Optional `?status=GENERATING|VALIDATING|ACCEPTED|FAILED_FINAL`.
- `GET /api/queries/{id}` — one request (including every attempt and every validation result).
- `GET /api/queries/sse` — server-sent events stream of every request change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "SQL Generator"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a question, live list of requests with status pills, click-to-expand per-attempt timeline showing each generated SQL, the guardrail verdict, the validator's verdict, and the validator's notes.

Browser title: `<title>Akka Sample: SQL Generator</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — mutation guardrail** (`before-tool-call`): a deterministic scan for forbidden mutating SQL keywords (`DROP`, `DELETE`, `UPDATE`, `INSERT`, `TRUNCATE`, `ALTER`). Queries containing any of these are short-circuited back to the GeneratorAgent with a structured feedback note (`reasonCode = MUTATION_FORBIDDEN`); they never reach the ValidatorAgent. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's validation result is recorded as a `ValidationEvalRecorded` event with `{ attemptNumber, verdict, score, mutationBlocked }`. The `EvalSampler` TimedAction is the canonical writer; the workflow also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/queries/{id}`.

## 9. Agent prompts

- `GeneratorAgent` → `prompts/generator.md`. Translates a natural-language question into a `SELECT`-only SQL query targeting the named schema; on a revision call, takes the prior `ValidationNotes` as input and produces a corrected query.
- `ValidatorAgent` → `prompts/validator.md`. Resolves every table and column reference against the schema definition, checks join conditions, runs a simulated `EXPLAIN`; returns `VALID` with a one-line rationale or `INVALID` with up to three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a natural-language question; request progresses `GENERATING` → `VALIDATING` → `ACCEPTED` within the retry ceiling; the App UI shows every attempt's SQL and validation result.
2. **J2 — halt at ceiling** — Submit a question whose schema references are impossible (test mode forces the Validator to `INVALID` every attempt); request progresses through every attempt and lands in `FAILED_FINAL` with the best-scoring query preserved and a structured failure reason.
3. **J3 — mutation guardrail block** — Submit a question that causes the GeneratorAgent to produce a query containing `DROP TABLE`; the guardrail short-circuits with `reasonCode = MUTATION_FORBIDDEN`, the GeneratorAgent re-drafts a `SELECT`-only query, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any completed request shows one `ValidationEvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sql-gen-validator-loop demonstrating the evaluator-optimizer ×
dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-dev-code-sql-gen-validator-loop.
Java package io.akka.samples.sqlgenerator. Akka 3.6.0. HTTP port 9335.

Components to wire (exactly):
- 2 AutonomousAgents:
  * GeneratorAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_QUERY).maxIterationsPerTask(3)).
    System prompt loaded from prompts/generator.md. Returns GeneratedQuery{sql,
    dialect, generatedAt} for both GENERATE and REVISE_QUERY. The
    REVISE_QUERY task takes (originalQuestion, schemaName, priorQuery,
    ValidationNotes) as inputs.
  * ValidatorAgent — definition() with
    capability(TaskAcceptance.of(VALIDATE).maxIterationsPerTask(2)). System
    prompt from prompts/validator.md. Returns ValidationResult{verdict, notes,
    score, evaluatedAt} where verdict is the ValidatorVerdict enum (VALID |
    INVALID) and score is a 1–5 integer rubric.

- 1 Workflow SqlWorkflow with steps:
    startStep -> generateStep -> guardrailStep -> [guardrail FAIL? generateStep
    again with structured feedback : validateStep] ->
    [verdict VALID? acceptStep : (attemptCount < maxAttempts ?
       generateStep with notes attached : failStep)] -> END.
  generateStep calls forAutonomousAgent(GeneratorAgent.class, requestId)
    .runSingleTask(GENERATE or REVISE_QUERY) then forTask(taskId)
    .result(GENERATE or REVISE_QUERY). validateStep calls
    forAutonomousAgent(ValidatorAgent.class, requestId).runSingleTask(VALIDATE).
    acceptStep emits QueryRequestAccepted. failStep emits
    QueryRequestFailedFinal with the highest-scoring attempt's SQL as
    best-of and a structured failureReason. Override settings() with
    stepTimeout(60s) on generateStep and validateStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(failStep)).
  guardrailStep is a pure-function step (no LLM call): scans
    query.sql() for any of {DROP, DELETE, UPDATE, INSERT, TRUNCATE, ALTER}
    as whole words, case-insensitive. On FAIL, emits
    AttemptMutationGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "MUTATION_FORBIDDEN", then transitions back to generateStep
    with a fixed ValidationNotes("Query contains a mutating keyword; rewrite
    as a SELECT-only statement."). The GeneratorAgent's next call sees the
    structured feedback as prior notes. The cycle still counts toward
    maxAttempts.

- 1 EventSourcedEntity QueryRequestEntity holding state QueryRequest{
  requestId, question, schemaName, maxAttempts, QueryStatus status,
  List<QueryAttempt> attempts, Optional<Integer> acceptedAttemptNumber,
  Optional<String> acceptedSql, Optional<String> failureReason,
  Instant createdAt, Optional<Instant> finishedAt}. QueryStatus enum:
  GENERATING, VALIDATING, ACCEPTED, FAILED_FINAL. Events:
  QueryRequestCreated, AttemptGenerated,
  AttemptMutationGuardrailVerdictRecorded, AttemptValidated,
  QueryRequestAccepted, QueryRequestFailedFinal, ValidationEvalRecorded.
  Commands: createRequest, recordGeneration, recordGuardrail,
  recordValidation, accept, failFinal, recordEval, getRequest.
  emptyState() returns QueryRequest.initial("", "", 4) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity RequestQueue with command enqueueQuestion(text,
  schemaName, requestedBy) emitting QuestionSubmitted{requestId, text,
  schemaName, requestedBy, submittedAt}.

- 1 View QueryRequestView with row type QueryRequestRow (mirrors
  QueryRequest; the attempts list is preserved as-is — the list is bounded
  at maxAttempts so size stays reasonable). Table updater consumes
  QueryRequestEntity events. ONE query getAllRequests SELECT * AS requests
  FROM query_request_view. No WHERE status filter — caller filters
  client-side because Akka cannot auto-index enum columns (Lesson 2).

- 1 Consumer SubmissionConsumer subscribed to RequestQueue events; on
  QuestionSubmitted starts a SqlWorkflow with the requestId as the
  workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sql-questions.jsonl and calls
    RequestQueue.enqueueQuestion.
  * EvalSampler — every 30s, queries QueryRequestView.getAllRequests, finds
    requests with a validated attempt that has not yet been recorded as a
    ValidationEvalRecorded event, and calls
    QueryRequestEntity.recordEval(attemptNumber, verdict, score,
    mutationBlocked). Idempotent per (requestId, attemptNumber).

- 2 HttpEndpoints:
  * SqlEndpoint at /api with POST /queries, GET /queries,
    GET /queries/{id}, GET /queries/sse, and three /api/metadata/*
    endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /queries body is
    {question, schemaName?, requestedBy?}; missing schemaName defaults
    to "default_schema", missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- SqlTasks.java declaring three Task<R> constants: GENERATE (resultConformsTo
  GeneratedQuery), REVISE_QUERY (GeneratedQuery), VALIDATE (ValidationResult).
- Domain records GeneratedQuery, MutationGuardrailVerdict, ValidationNotes,
  ValidationResult, QueryAttempt, QueryRequest; enums QueryStatus,
  ValidatorVerdict.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9335 and
  akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  sql-gen.workflow.max-attempts = 4 and
  sql-gen.workflow.default-schema = "default_schema", overridable by
  env var.
- src/main/resources/sample-events/sql-questions.jsonl with 8 canned
  question lines, each shaped {"text":"...", "schemaName":"orders_schema"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 mutation
  guardrail before-tool-call, E1 eval-event on-decision-eval) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = sql-generation-from-natural-language,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/generator.md, prompts/validator.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: SQL Generator",
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
  <title>Akka Sample: SQL Generator</title>.

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
  named in Section 9: generator.json, validator.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    generator.json — 6 GeneratedQuery entries. Three are valid SELECT
      queries against the sample schemas (orders, products, customers).
      Two are revised SELECT queries that reference the prior validation
      notes (fixing a missing JOIN or wrong column name). One is an
      intentionally mutating query containing "DELETE FROM orders" used
      to exercise the mutation guardrail in J3.
    validator.json — 6 ValidationResult entries. Three return
      verdict=VALID with score=4 or 5 and a one-sentence rationale.
      Three return verdict=INVALID with score=2 or 3 and a
      ValidationNotes payload of three bullets ("column 'qty' not found
      in table 'products'", "missing JOIN condition between orders and
      customers", "ambiguous column reference 'id'").
- A MockModelProvider.seedFor(requestId, attemptNumber) helper makes the
  selection deterministic per (requestId, attemptNumber) so the same
  request in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
  GeneratorAgent and ValidatorAgent both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and ship with a SqlTasks
  companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the QueryRequest row record
  is Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: SqlTasks.java is mandatory; generating GeneratorAgent or
  ValidatorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9335, declared in application.conf
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
