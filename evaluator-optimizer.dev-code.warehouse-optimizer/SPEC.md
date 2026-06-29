# SPEC — warehouse-optimizer

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Data Warehouse Optimizer.
**One-line pitch:** Submit a slow query or a schema change request; an optimizer agent proposes the improvement; an evaluator agent scores it against cost, correctness, and DDL-safety rules; the two iterate until the evaluator approves or the loop hits its attempt ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`OptimizerAgent`) and a reviewer agent (`EvaluatorAgent`), feeding each evaluation back into the next proposal until convergence or a halt. The blueprint also demonstrates four governance mechanisms — a **guardrail** that blocks DDL proposals before the evaluator runs (warehouse DDL is destructive and cannot be scored the same way as query rewrites), a **human-in-the-loop** gate that pauses the workflow for DBA approval on any DDL proposal before it proceeds, an **eval-event** that records every cycle's verdict for downstream quality measurement, and a **halt** that ends the loop gracefully at the retry ceiling.

## 3. User-facing flows

The user opens the App UI tab and submits an optimization request (an original query or DDL statement plus an objective such as "reduce scan rows by at least 50%").

1. The system creates an `OptimizationRequest` record in `PROPOSING` and starts an `OptimizationWorkflow`.
2. The Optimizer proposes attempt #1: a rewritten query or schema change with a brief rationale.
3. The DDL guardrail inspects the proposal. If it contains DDL keywords (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME`), the guardrail fires (`reasonCode = DDL_DETECTED`) and the workflow pauses to wait for DBA approval before continuing. Non-DDL proposals pass immediately.
4. On DDL proposals that pass the DBA gate, or on non-DDL proposals, the Evaluator scores the proposal against a rubric (estimated cost reduction, semantic correctness, safety, and side-effect risk) and returns either `APPROVE` with a one-line rationale, or `REVISE` with a typed `EvaluationNotes` payload (three bullets at most).
5. On `APPROVE`, the workflow transitions the request to `APPROVED` with the winning proposal's text and the evaluator's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the DBA decision (if any), and the evaluation on the entity, then calls the Optimizer again with the evaluation attached. The Optimizer produces attempt #2.
7. If the loop reaches `maxAttempts` (default 4) without an `APPROVE`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring proposal is preserved on the entity along with every evaluation for audit, and an `EvalRecorded` event captures the rejection.

A `RequestSimulator` (TimedAction) drips a canned optimization request every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OptimizerAgent` | `AutonomousAgent` | Proposes a query rewrite or schema change; accepts prior evaluation on revisions. | `OptimizationWorkflow` | returns `Proposal` to workflow |
| `EvaluatorAgent` | `AutonomousAgent` | Scores a proposal against the rubric; returns `APPROVE` or `REVISE` with notes. | `OptimizationWorkflow` | returns `Evaluation` to workflow |
| `OptimizationWorkflow` | `Workflow` | Runs the propose → guardrail → HITL → evaluate → revise loop; halts at the ceiling. | `OptimizerEndpoint`, `RequestConsumer` | `OptimizationRequestEntity` |
| `OptimizationRequestEntity` | `EventSourcedEntity` | Holds the request lifecycle, every proposal, every evaluation, and the final outcome. | `OptimizationWorkflow` | `RequestsView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted request for replay and audit. | `OptimizerEndpoint`, `RequestSimulator` | `RequestConsumer` |
| `RequestsView` | `View` | List-of-requests read model. | `OptimizationRequestEntity` events | `OptimizerEndpoint` |
| `RequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a workflow per submission. | `RequestQueue` events | `OptimizationWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample request every 60 s from `sample-events/warehouse-requests.jsonl`. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `RequestsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `OptimizationRequestEntity` |
| `DBAApprovalGate` | `TimedAction` | Polls `RequestsView` for requests awaiting DBA approval and exposes a `/api/requests/{id}/dba-decision` endpoint action (via `OptimizerEndpoint`) for human confirmation. | scheduler + HTTP | `OptimizationWorkflow` (resume signal) |
| `OptimizerEndpoint` | `HttpEndpoint` | `/api/requests/*` — submit, get, list, SSE, DBA decision; plus `/api/metadata/*`. | — | `RequestsView`, `RequestQueue`, `OptimizationRequestEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record OptimizationTarget(String originalSql, String objective, String submittedBy) {}

record Proposal(
    String proposedSql,
    ProposalKind kind,
    String rationale,
    Instant proposedAt
) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record DBADecision(boolean approved, String decisionNote, String decidedBy, Instant decidedAt) {}

record EvaluationNotes(List<String> bullets, String overallRationale) {}

record Evaluation(
    EvaluatorVerdict verdict,
    EvaluationNotes notes,
    int score,
    Instant evaluatedAt
) {}

record Attempt(
    int attemptNumber,
    Proposal proposal,
    GuardrailVerdict guardrail,
    Optional<DBADecision> dbaDecision,
    Optional<Evaluation> evaluation
) {}

record OptimizationRequest(
    String requestId,
    String originalSql,
    String objective,
    int maxAttempts,
    RequestStatus status,
    List<Attempt> attempts,
    Optional<Integer> approvedAttemptNumber,
    Optional<String> approvedSql,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus { PROPOSING, AWAITING_DBA, EVALUATING, APPROVED, REJECTED_FINAL }

enum EvaluatorVerdict { APPROVE, REVISE }

enum ProposalKind { QUERY_REWRITE, INDEX_ADD, PARTITION_PRUNE, DDL_ALTER, DDL_CREATE, DDL_DROP }
```

### Events (on `OptimizationRequestEntity`)

`RequestCreated`, `AttemptProposed`, `AttemptGuardrailVerdictRecorded`, `DBADecisionRecorded`, `AttemptEvaluated`, `RequestApproved`, `RequestRejectedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/requests` — body `{ originalSql, objective?, submittedBy? }` → `{ requestId }`. Starts a workflow.
- `GET /api/requests` — list all requests. Optional `?status=PROPOSING|AWAITING_DBA|EVALUATING|APPROVED|REJECTED_FINAL`.
- `GET /api/requests/{id}` — one request (including every attempt and every evaluation).
- `GET /api/requests/sse` — server-sent events stream of every request change.
- `POST /api/requests/{id}/dba-decision` — body `{ approved, decisionNote?, decidedBy? }` → `200`. Resumes a paused workflow.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Data Warehouse Optimizer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (guardrail = red, hitl = orange, eval-event = blue, halt = red).
- **App UI** — form to submit an optimization request, live list of requests with status pills, click-to-expand per-attempt timeline showing each proposal, the guardrail verdict, the DBA decision (if applicable), the evaluator's verdict, and the evaluator's notes. A DBA decision form appears inline for requests in `AWAITING_DBA`.

Browser title: `<title>Akka Sample: Data Warehouse Optimizer</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — DDL guardrail** (`before-tool-call`): a deterministic check on the proposal text before the evaluator is called. If the proposal contains DDL keywords (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME`), the guardrail fires (`reasonCode = DDL_DETECTED`), records `AttemptGuardrailVerdictRecorded` with `verdict.passed = false`, and transitions the workflow to the DBA approval pause state. Non-DDL proposals pass with `reasonCode = OK`. Enforcement: blocking.
- **H1 — DBA approval gate** (`application`): when the DDL guardrail fires, the workflow transitions the request to `AWAITING_DBA` and waits for a human DBA decision via `POST /api/requests/{id}/dba-decision`. If the DBA approves, the workflow resumes and calls the Evaluator. If the DBA rejects, the workflow ends with `REJECTED_FINAL` immediately — the DDL never executes. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's evaluation is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, ddlDetected }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxAttempts` without an `APPROVE`, the workflow ends with `REJECTED_FINAL`. The entity preserves every proposal, every evaluation, the best-scoring proposal's SQL, and a structured rejection reason. Enforcement: system-level.

## 9. Agent prompts

- `OptimizerAgent` → `prompts/optimizer.md`. Proposes a query rewrite or schema change on the original SQL and objective; on a revision call, takes the prior `EvaluationNotes` as input and produces a new proposal.
- `EvaluatorAgent` → `prompts/evaluator.md`. Scores a proposal against the fixed rubric; returns `APPROVE` with a one-line rationale or `REVISE` with three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a query-rewrite request; request progresses `PROPOSING` → `EVALUATING` → `APPROVED` within the retry ceiling; the App UI shows every attempt's proposal and evaluation.
2. **J2 — halt at ceiling** — Submit a request whose rubric is impossible (test mode forces the Evaluator to `REVISE` every attempt); request progresses through every attempt and lands in `REJECTED_FINAL` with the best proposal preserved and a structured rejection reason.
3. **J3 — DDL guardrail and DBA gate** — Submit a DDL request; the guardrail fires with `reasonCode = DDL_DETECTED`; the request enters `AWAITING_DBA`; the DBA decision form appears; submitting approval resumes the workflow; the Evaluator then scores the proposal normally.
4. **J4 — eval-event timeline** — The expanded view of any completed request shows one `EvalRecorded` event per critiqued attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named warehouse-optimizer demonstrating the evaluator-optimizer ×
dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-dev-code-warehouse-optimizer.
Java package io.akka.samples.datawarehouseoptimizer. Akka 3.6.0. HTTP port 9771.

Components to wire (exactly):
- 2 AutonomousAgents:
  * OptimizerAgent — definition() with
    capability(TaskAcceptance.of(PROPOSE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_PROPOSAL).maxIterationsPerTask(3)).
    System prompt loaded from prompts/optimizer.md. Returns Proposal{proposedSql,
    kind, rationale, proposedAt} for both PROPOSE and REVISE_PROPOSAL. The
    REVISE_PROPOSAL task takes (originalSql, objective, priorProposal,
    EvaluationNotes) as inputs.
  * EvaluatorAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE).maxIterationsPerTask(2)). System
    prompt from prompts/evaluator.md. Returns Evaluation{verdict, notes, score,
    evaluatedAt} where verdict is the EvaluatorVerdict enum (APPROVE | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow OptimizationWorkflow with steps:
    startStep -> proposeStep -> guardrailStep ->
    [guardrail DDL? dbaGateStep : evaluateStep] ->
    [dbaGateStep: await DBADecisionRecorded, if approved -> evaluateStep,
     if rejected -> rejectStep] ->
    [verdict APPROVE? approveStep : (attemptCount < maxAttempts ?
       proposeStep with evaluation attached : rejectStep)] -> END.
  proposeStep calls forAutonomousAgent(OptimizerAgent.class, requestId)
    .runSingleTask(PROPOSE or REVISE_PROPOSAL) then
    forTask(taskId).result(PROPOSE or REVISE_PROPOSAL).
  evaluateStep calls forAutonomousAgent(EvaluatorAgent.class, requestId)
    .runSingleTask(EVALUATE).
  approveStep emits RequestApproved.
  rejectStep emits RequestRejectedFinal with the highest-scoring proposal's
    SQL as best-of and a structured rejectionReason.
  dbaGateStep pauses the workflow (no LLM call); it emits
    AttemptGuardrailVerdictRecorded (passed=false, DDL_DETECTED) and
    transitions the request to AWAITING_DBA. The workflow resumes on the
    DBADecisionRecorded command delivered via POST /api/requests/{id}/dba-decision.
  guardrailStep is a pure-function step (no LLM call): checks whether the
    proposal.proposedSql contains DDL keywords (case-insensitive: CREATE,
    ALTER, DROP, TRUNCATE, RENAME). On DDL detected, emits
    AttemptGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "DDL_DETECTED", then transitions to dbaGateStep.
    On clean, emits verdict.passed = true (reasonCode = "OK") and advances
    to evaluateStep.
  Override settings() with stepTimeout(60s) on proposeStep and evaluateStep,
    and defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).

- 1 EventSourcedEntity OptimizationRequestEntity holding state
  OptimizationRequest{requestId, originalSql, objective, maxAttempts,
  RequestStatus status, List<Attempt> attempts,
  Optional<Integer> approvedAttemptNumber, Optional<String> approvedSql,
  Optional<String> rejectionReason, Instant createdAt,
  Optional<Instant> finishedAt}. RequestStatus enum: PROPOSING,
  AWAITING_DBA, EVALUATING, APPROVED, REJECTED_FINAL.
  Events: RequestCreated, AttemptProposed, AttemptGuardrailVerdictRecorded,
  DBADecisionRecorded, AttemptEvaluated, RequestApproved,
  RequestRejectedFinal, EvalRecorded.
  Commands: createRequest, recordProposal, recordGuardrail, recordDBADecision,
  recordEvaluation, approve, rejectFinal, recordEval, getRequest.
  emptyState() returns OptimizationRequest.initial("", "", "", 4) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity RequestQueue with command enqueueRequest(originalSql,
  objective, submittedBy) emitting RequestSubmitted{requestId, originalSql,
  objective, submittedBy, submittedAt}.

- 1 View RequestsView with row type RequestRow (mirrors OptimizationRequest;
  the attempts list is preserved as-is — the list is bounded at maxAttempts
  so size stays reasonable). Table updater consumes OptimizationRequestEntity
  events. ONE query getAllRequests SELECT * AS requests FROM requests_view.
  No WHERE status filter — caller filters client-side because Akka cannot
  auto-index enum columns (Lesson 2).

- 1 Consumer RequestConsumer subscribed to RequestQueue events; on
  RequestSubmitted starts an OptimizationWorkflow with the requestId as the
  workflow id.

- 3 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/warehouse-requests.jsonl and calls
    RequestQueue.enqueueRequest.
  * EvalSampler — every 30s, queries RequestsView.getAllRequests, finds
    requests with an evaluated attempt that has not yet been recorded as an
    EvalRecorded event, and calls OptimizationRequestEntity.recordEval(
    attemptNumber, verdict, score, ddlDetected). Idempotent per
    (requestId, attemptNumber).
  * DBAApprovalGate — every 15s, queries RequestsView.getAllRequests for
    requests in AWAITING_DBA status older than 5 minutes, logs a reminder
    note to the service log. (Actual DBA decisions are delivered via the
    HTTP endpoint.)

- 2 HttpEndpoints:
  * OptimizerEndpoint at /api with POST /requests, GET /requests,
    GET /requests/{id}, GET /requests/sse,
    POST /requests/{id}/dba-decision, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
    The POST /requests body is {originalSql, objective?, submittedBy?};
    missing objective defaults to "improve performance", missing submittedBy
    defaults to "anonymous". The POST /requests/{id}/dba-decision body is
    {approved, decisionNote?, decidedBy?}; missing decidedBy defaults to
    "dba".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- WarehouseTasks.java declaring three Task<R> constants: PROPOSE (resultConformsTo
  Proposal), REVISE_PROPOSAL (Proposal), EVALUATE (Evaluation).
- Domain records Proposal, GuardrailVerdict, DBADecision, EvaluationNotes,
  Evaluation, Attempt, OptimizationRequest; enums RequestStatus,
  EvaluatorVerdict, ProposalKind.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9771 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read
  from the canonical env vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  warehouse-optimizer.optimization.max-attempts = 4 and
  warehouse-optimizer.optimization.dba-gate-timeout-minutes = 60,
  overridable by env var.
- src/main/resources/sample-events/warehouse-requests.jsonl with 8 canned
  request lines, each shaped
  {"originalSql":"SELECT ...","objective":"reduce scan rows by 50%"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 4 controls (G1 DDL guardrail
  before-tool-call, H1 DBA HITL application, E1 eval-event on-decision-eval,
  HT1 halt graceful-degradation) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = query-optimization,
  decisions.authority_level = recommendation-only,
  data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/optimizer.md, prompts/evaluator.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Data Warehouse Optimizer",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customize-before-generating, what-gets-validated, license.
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
  rows, mechanism badges: guardrail=red, hitl=orange, eval-event=blue,
  halt=red); App UI (form + live list with status pills, click-to-expand
  per-attempt timeline; DBA decision form appears inline for AWAITING_DBA
  requests). Browser title exactly:
  <title>Akka Sample: Data Warehouse Optimizer</title>.

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
  named in Section 9: optimizer.json, evaluator.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    optimizer.json — 6 Proposal entries. Three are query-rewrite proposals
      (ProposalKind.QUERY_REWRITE) with rewritten SELECT statements adding
      index hints or subquery pushdowns. Two are revision proposals that
      reference the prior evaluation (more targeted rewrites). One is a
      DDL proposal (ProposalKind.DDL_ALTER) that adds an index, used to
      exercise the DDL guardrail and DBA gate in J3.
    evaluator.json — 6 Evaluation entries. Three return verdict=APPROVE
      with score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and an EvaluationNotes payload of
      three bullets ("scan row estimate still exceeds target", "missing
      covering index for ORDER BY clause", "subquery correlated — rewrite
      as JOIN").
- A MockModelProvider.seedFor(requestId, attemptNumber) helper makes the
  selection deterministic per (requestId, attemptNumber) so the same
  request in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. OptimizerAgent
  and EvaluatorAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a WarehouseTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the OptimizationRequest row
  record is Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: WarehouseTasks.java is mandatory; generating OptimizerAgent or
  EvaluatorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9771, declared in application.conf
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
