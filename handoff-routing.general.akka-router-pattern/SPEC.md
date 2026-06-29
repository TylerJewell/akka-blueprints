# SPEC — akka-router-pattern

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** OpenAI Agents Routing Pattern.
**One-line pitch:** A classifier agent examines an incoming task request, a before-invocation guardrail checks the classified request, and the request is handed off to the appropriate specialist agent that executes it end-to-end — with a durable workflow recording both the routing decision and the specialist result.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *who* should own the task, then a guardrail verifies the routed request is safe to hand off, and the downstream specialist agent receives the request and produces the final result. The downstream agent is responsible for the whole execution; the classifier does not assist.

Two governance mechanisms are layered on top:

- A **before-agent-invocation guardrail** runs inside the `RouterWorkflow` after the routing decision and before the specialist is called. It inspects the classified request for prompt-injection patterns, out-of-scope content, and policy violations. If the check fails the request is blocked before any specialist sees it.
- An **on-decision eval** fires every time the classifier emits a routing decision. A `RoutingJudge` agent grades the decision against the original request on a 1–5 rubric. The score and rationale are written back to the request entity and surfaced in the UI.

The pattern is a textbook classifier-then-execute fan-out: the workflow branches on the classifier's domain, and only the chosen specialist is invoked. The other specialists see no traffic for that request.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live request list. Every request displays its domain chip, status pill, routing score, and (if completed) the published result.
2. `TaskSimulator` (TimedAction) ticks every 30 s and inserts a new canned task from `sample-events/task-requests.jsonl` into `TaskQueue`.
3. For each new request: `RouterWorkflow` is started by a Consumer reading `TaskQueue` events, which also registers a `RequestEntity`.
4. The workflow calls `ClassifierAgent`, gets a `ClassificationDecision { domain, confidence, reason }`, and emits `ClassificationDecided` on the entity.
5. The workflow calls `RoutingGuardrail` with the `ClassificationDecision` and the original `TaskRequest`. If the verdict is `denied`, the workflow emits `RequestBlocked` and ends.
6. Branch on `domain`:
   - `CONTENT` → workflow calls `ContentSpecialist` with the `EXECUTE` task.
   - `CODE` → workflow calls `CodeSpecialist` with the `EXECUTE` task.
   - `DATA` → workflow calls `DataSpecialist` with the `EXECUTE` task.
   - `UNKNOWN` → workflow emits `RequestUnroutable`; ends.
7. The specialist returns a typed `TaskResult`. The workflow emits `ResultPublished` (terminal `COMPLETED`).
8. Independent of the workflow, `RoutingEvalScorer` (Consumer) listens for `ClassificationDecided` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the request entity.
9. The user can click any request card and see the original task, the classification decision, the routing score, the chosen specialist, and the published result.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskSimulator` | `TimedAction` | Drips simulated task requests into `TaskQueue` every 30 s. | scheduler | `TaskQueue` |
| `TaskQueue` | `EventSourcedEntity` | Append-only audit log of every inbound task request (`InboundTaskReceived`). | `TaskSimulator`, `RouterEndpoint` | `QueueConsumer` |
| `QueueConsumer` | `Consumer` | Subscribes to `TaskQueue` events; registers `RequestEntity`; starts a `RouterWorkflow`. | `TaskQueue` events | `RequestEntity`, `RouterWorkflow` |
| `ClassifierAgent` | `Agent` (typed, not autonomous) | Classifies a `TaskRequest` into `CONTENT` / `CODE` / `DATA` / `UNKNOWN` with confidence + reason. | invoked by `RouterWorkflow` | returns `ClassificationDecision` |
| `RoutingGuardrail` | `Agent` (typed) | Before-agent-invocation guardrail: checks the classified request for injection patterns and policy violations. Returns `GuardrailVerdict { verdict, violations }`. | invoked by `RouterWorkflow` | returns `GuardrailVerdict` |
| `ContentSpecialist` | `AutonomousAgent` | Owns the `EXECUTE` task for content-domain requests. Returns typed `TaskResult`. | invoked by `RouterWorkflow` | returns `TaskResult` |
| `CodeSpecialist` | `AutonomousAgent` | Owns the `EXECUTE` task for code-domain requests. Returns typed `TaskResult`. | invoked by `RouterWorkflow` | returns `TaskResult` |
| `DataSpecialist` | `AutonomousAgent` | Owns the `EXECUTE` task for data-domain requests. Returns typed `TaskResult`. | invoked by `RouterWorkflow` | returns `TaskResult` |
| `RoutingJudge` | `Agent` (typed) | Grades a classification decision against the original task request. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RoutingEvalScorer` | returns `RoutingScore` |
| `RouterWorkflow` | `Workflow` | Per-request orchestration: classify → guardrail → route → execute → publish. | `QueueConsumer` (start) | `RequestEntity` |
| `RequestEntity` | `EventSourcedEntity` | Per-request lifecycle. | `RouterWorkflow`, `RoutingEvalScorer` | `RequestView` |
| `RequestView` | `View` | Read-model row per request. | `RequestEntity` events | `RouterEndpoint` |
| `RoutingEvalScorer` | `Consumer` | Subscribes to `RequestEntity` events; on `ClassificationDecided` invokes `RoutingJudge` and writes `RoutingScored` back. | `RequestEntity` events | `RequestEntity` |
| `RouterEndpoint` | `HttpEndpoint` | `/api/requests/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `RequestView`, `RequestEntity`, `TaskQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record TaskRequest(
    String requestId,
    String requesterId,
    String channel,            // "api" | "web-form" | "cli"
    String title,
    String body,
    Instant receivedAt
) {}

enum TaskDomain { CONTENT, CODE, DATA, UNKNOWN }

record ClassificationDecision(
    TaskDomain domain,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

enum GuardrailOutcome { ALLOWED, DENIED }

record GuardrailVerdict(
    GuardrailOutcome verdict,
    List<String> violations,   // empty when ALLOWED
    String rubricVersion
) {}

enum TaskResultStatus { COMPLETED, ESCALATED }

record TaskResult(
    String resultTitle,
    String resultBody,
    TaskResultStatus status,
    String specialistTag,      // "content" | "code" | "data"
    Instant completedAt
) {}

record RoutingScore(
    int score,                 // 1..5
    String rationale,
    Instant scoredAt
) {}

record Request(
    String requestId,
    TaskRequest incoming,
    Optional<ClassificationDecision> classification,
    Optional<GuardrailVerdict> guardrail,
    Optional<TaskResult> result,
    Optional<RoutingScore> routingScore,
    Optional<String> unroutableReason,
    RequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus {
    RECEIVED,
    CLASSIFIED,
    GUARDRAIL_PASSED,
    ROUTED_CONTENT,
    ROUTED_CODE,
    ROUTED_DATA,
    EXECUTING,
    BLOCKED,
    COMPLETED,
    UNROUTABLE
}
```

Events on `RequestEntity`: `RequestRegistered`, `ClassificationDecided`, `GuardrailVerdictAttached`, `RequestRouted`, `ExecutionStarted`, `RequestBlocked`, `ResultPublished`, `RequestUnroutable`, `RoutingScored`.

Events on `TaskQueue`: `InboundTaskReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/requests` — list all requests (newest-first), optional `?domain=CONTENT|CODE|DATA|UNKNOWN&status=…` filtered client-side.
- `GET /api/requests/{id}` — one request.
- `POST /api/requests` — manually submit a task request (body `TaskRequest` minus `requestId` and `receivedAt`); server assigns both.
- `POST /api/requests/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `BLOCKED` to `COMPLETED` using the specialist that was designated before the guardrail fired.
- `GET /api/requests/sse` — Server-Sent Events for every request change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: OpenAI Agents Routing Pattern</title>`.

The App UI tab is a three-pane layout: **left** is the request list (status pill + domain chip + score chip), **centre** is the selected request's task body + classification decision + score, **right** is the chosen specialist's result + guardrail verdict (or violations + Unblock button when `BLOCKED`, or "Unroutable" notice when `UNROUTABLE`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-invocation guardrail** (`RoutingGuardrail` Agent): checks the classified request against a rubric (no prompt-injection tokens, no requests for out-of-scope personal data access, no cross-domain confusion where a code request is tagged as content). Blocking — a violation puts the request in `BLOCKED` before any specialist is invoked.
- **E1 — on-decision eval** (`eval-event`, on the classification decision): `RoutingEvalScorer` (Consumer) listens for `ClassificationDecided` events and calls `RoutingJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate; persistent low scores signal a regression in routing quality.

## 9. Agent prompts

- `ClassifierAgent` → `prompts/classifier-agent.md`. Typed classifier; returns one of `CONTENT`, `CODE`, `DATA`, `UNKNOWN`; defaults to `UNKNOWN` under ambiguity.
- `RoutingGuardrail` → `prompts/routing-guardrail.md`. Before-agent-invocation check. Returns `GuardrailVerdict { verdict, violations, rubricVersion }`. Conservative — borderline requests are blocked.
- `ContentSpecialist` → `prompts/content-specialist.md`. Owns the `EXECUTE` task for content-domain requests.
- `CodeSpecialist` → `prompts/code-specialist.md`. Owns the `EXECUTE` task for code-domain requests.
- `DataSpecialist` → `prompts/data-specialist.md`. Owns the `EXECUTE` task for data-domain requests.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a classification decision against a 1–5 rubric with one-sentence rationale.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a content-domain request → classified `CONTENT` → guardrail passes → executed by `ContentSpecialist` → published.
2. **J2** — Simulator drips a code request → classified `CODE` → guardrail passes → executed by `CodeSpecialist` → published.
3. **J3** — A data-domain request classifies as `DATA`, guardrail passes, `DataSpecialist` produces the result.
4. **J4** — An ambiguous request classifies as `UNKNOWN` and lands in `UNROUTABLE` without any specialist invocation.
5. **J5** — A request containing a prompt-injection pattern trips the guardrail; the request lands in `BLOCKED` with the violation listed.
6. **J6** — Every classified request carries a `RoutingScore` (1–5) and rationale within ~10 s of the classification decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-router-pattern demonstrating the handoff-routing × general cell.
Runs out of the box (in-process simulated inbound stream; no real external integration).
Maven group io.akka.samples. Maven artifact handoff-routing-general-akka-router-pattern. Java package
io.akka.samples.openaiagentsroutingpattern. Akka 3.6.0. HTTP port 9866.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ClassifierAgent — classifier. System prompt loaded from
  prompts/classifier-agent.md. Input: TaskRequest{requestId, requesterId, channel, title,
  body, receivedAt}. Output: ClassificationDecision{domain: TaskDomain
  (CONTENT/CODE/DATA/UNKNOWN), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNKNOWN under uncertainty.

- 1 Agent (typed) RoutingGuardrail — before-agent-invocation guardrail. System prompt from
  prompts/routing-guardrail.md. Input: TaskRequest + ClassificationDecision. Output:
  GuardrailVerdict{verdict: GuardrailOutcome (ALLOWED/DENIED), violations: List<String>,
  rubricVersion: String}. Used by RouterWorkflow before routing; blocking.

- 1 AutonomousAgent ContentSpecialist — definition() with capability(TaskAcceptance.of(EXECUTE)
  .maxIterationsPerTask(3)). System prompt from prompts/content-specialist.md. Input:
  TaskRequest + ClassificationDecision. Output: TaskResult{resultTitle, resultBody,
  status: TaskResultStatus, specialistTag = "content", completedAt}. Sets
  status=ESCALATED when outside capability.

- 1 AutonomousAgent CodeSpecialist — definition() with capability(TaskAcceptance.of(EXECUTE)
  .maxIterationsPerTask(3)). System prompt from prompts/code-specialist.md. Same
  input shape; specialistTag = "code".

- 1 AutonomousAgent DataSpecialist — definition() with capability(TaskAcceptance.of(EXECUTE)
  .maxIterationsPerTask(3)). System prompt from prompts/data-specialist.md. Same
  input shape; specialistTag = "data".

- 1 Agent (typed) RoutingJudge — judge. System prompt from prompts/routing-judge.md. Input:
  TaskRequest + ClassificationDecision. Output: RoutingScore{score: int 1–5, rationale: String,
  scoredAt: Instant}.

- 1 Workflow RouterWorkflow per requestId. Steps:
    classifyStep -> guardrailStep -> routeStep -> {contentStep | codeStep | dataStep | unroutableStep}
                -> publishStep
  classifyStep calls componentClient.forAgent().inSession(requestId).method(ClassifierAgent::classify)
    .invoke(request). On success emits ClassificationDecided via RequestEntity.recordClassification.
  guardrailStep calls componentClient.forAgent().inSession(requestId).method(RoutingGuardrail::check)
    .invoke(request, classification). On verdict.verdict=DENIED, emits RequestBlocked via
    RequestEntity.block (terminal BLOCKED) and ends. On ALLOWED, emits GuardrailVerdictAttached.
  routeStep branches on ClassificationDecision.domain:
    CONTENT -> proceed to contentStep (emits RequestRouted{CONTENT})
    CODE -> proceed to codeStep (emits RequestRouted{CODE})
    DATA -> proceed to dataStep (emits RequestRouted{DATA})
    UNKNOWN -> unroutableStep (emits RequestUnroutable; terminates).
  contentStep / codeStep / dataStep call forAutonomousAgent(<Specialist>.class, requestId)
    .runSingleTask(TaskDef.instructions(buildPrompt(request, classification))) returning a taskId,
    then forTask(taskId).result(RouterTasks.EXECUTE) to block on the typed TaskResult.
    On success emits ExecutionStarted then ResultPublished (terminal COMPLETED).
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on classifyStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on contentStep, codeStep, dataStep,
    and publishStep. defaultStepRecovery(maxRetries(2).failoverTo(RouterWorkflow::error)).

- 2 EventSourcedEntities:
    * TaskQueue — append-only audit log. Command receive(TaskRequest) emits
      InboundTaskReceived{request}. No mutable state beyond a counter; commands are
      idempotent on request.requestId.
    * RequestEntity (one per requestId) — full per-request lifecycle. State Request{requestId,
      incoming: TaskRequest, Optional<ClassificationDecision> classification,
      Optional<GuardrailVerdict> guardrail, Optional<TaskResult> result,
      Optional<RoutingScore> routingScore, Optional<String> unroutableReason,
      RequestStatus status, Instant createdAt, Optional<Instant> finishedAt}.
      RequestStatus enum: RECEIVED, CLASSIFIED, GUARDRAIL_PASSED, ROUTED_CONTENT,
      ROUTED_CODE, ROUTED_DATA, EXECUTING, BLOCKED, COMPLETED, UNROUTABLE.
      Events: RequestRegistered, ClassificationDecided, GuardrailVerdictAttached,
      RequestRouted, ExecutionStarted, RequestBlocked, ResultPublished,
      RequestUnroutable, RoutingScored. Commands: registerRequest, recordClassification,
      recordGuardrailVerdict, recordRouting, recordExecutionStart, block, publish,
      markUnroutable, recordRoutingScore, unblock, getRequest. emptyState() returns
      Request.initial("") with no commandContext() reference.

- 2 Consumers:
    * QueueConsumer subscribed to TaskQueue events; for each InboundTaskReceived
      calls RequestEntity.registerRequest for the requestId; then starts a RouterWorkflow
      with requestId as the workflow id.
    * RoutingEvalScorer subscribed to RequestEntity events; on ClassificationDecided invokes
      RoutingJudge.score(request, decision) and calls RequestEntity.recordRoutingScore(
      requestId, score). On any other event type, no-op. Use componentClient — do NOT
      call the agent from a TimedAction.

- 1 View RequestView with row type RequestRow (mirrors Request; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). Table updater consumes RequestEntity events.
  ONE query getAllRequests SELECT * AS requests FROM request_view. No WHERE domain or
  WHERE status filter — filter client-side in callers.

- 1 TimedAction TaskSimulator — every 30s, reads next line from
  src/main/resources/sample-events/task-requests.jsonl (loops at EOF) and calls
  TaskQueue.receive with a fresh requestId (UUID).

- 2 HttpEndpoints:
    * RouterEndpoint at /api with GET /requests (list from RequestView.getAllRequests,
      filter client-side by ?domain and ?status query params), GET /requests/{id},
      POST /requests (body TaskRequest minus requestId/receivedAt — server assigns),
      POST /requests/{id}/unblock (body {decidedBy, note} — operator override:
      re-routes the blocked request to the originally-intended specialist and publishes
      result as COMPLETED with an audit note),
      GET /requests/sse (serverSentEventsForView over getAllRequests), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- RouterTasks.java declaring the task constants: EXECUTE (resultConformsTo TaskResult.class,
  description "Execute the task request end-to-end and return a typed TaskResult").
- Domain records TaskRequest, ClassificationDecision, GuardrailVerdict, TaskResult,
  RoutingScore, and the Request entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9866 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/task-requests.jsonl with 9 canned lines (3 CONTENT-
  flavoured, 3 CODE-flavoured, 2 DATA-flavoured, 1 UNKNOWN-ambiguous, 1 designed to
  trip the guardrail with a prompt-injection pattern).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: G1 guardrail before-agent-invocation
  routing-safety, E1 eval-event on-decision-eval. Matching simplified_view list.
  No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = task-routing,
  decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false (the system executes without HITL by default — only
  blocked requests wait for a human), failure.failure_modes including "wrong-domain-routing",
  "prompt-injection-via-task-body", "cross-domain-output-confusion",
  "specialist-out-of-scope-execution"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/classifier-agent.md, prompts/routing-guardrail.md, prompts/content-specialist.md,
  prompts/code-specialist.md, prompts/data-specialist.md, prompts/routing-judge.md loaded
  as agent system prompts.
- README.md at the project root: title "Akka Sample: OpenAI Agents Routing Pattern",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = request list with domain chip + status pill + score chip; centre = task body +
  classification block; right = specialist result + guardrail verdict + published result
  or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: OpenAI Agents Routing Pattern</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call (seeded by requestId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    classifier-agent.json — 10 ClassificationDecision entries spanning CONTENT (copywriting,
      blog posts, email drafts), CODE (debug help, unit tests, refactoring),
      DATA (SQL queries, analysis scripts, aggregation tasks), and UNKNOWN
      (ambiguous mixed requests, very short messages). Confidence + one-sentence
      reason on each.
    routing-guardrail.json — 10 GuardrailVerdict entries. 7 with verdict=ALLOWED and
      empty violations. 3 with verdict=DENIED and one violation each
      ("prompt-injection-detected", "out-of-scope-pii-access", "cross-domain-confusion").
      The mock should match the guardrail-tripping entries above when called for the
      same requestId.
    content-specialist.json — 6 TaskResult entries: 5 with status COMPLETED (various
      writing tasks completed), 1 with status ESCALATED (request outside content scope).
    code-specialist.json — 6 TaskResult entries: 5 with status COMPLETED (debug results,
      unit tests, refactoring plans), 1 with status ESCALATED (outside programming scope).
    data-specialist.json — 6 TaskResult entries: 5 with status COMPLETED (SQL queries,
      analysis summaries), 1 with status ESCALATED (outside data analytics scope).
    routing-judge.json — 10 RoutingScore entries, score 1–5, one-sentence rationale
      matching the rubric (domain-correctness / confidence-calibration / reason-quality).
- A MockModelProvider.seedFor(requestId) helper makes per-request selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  ContentSpecialist, CodeSpecialist, and DataSpecialist all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): classifyStep 20s,
  guardrailStep 20s, contentStep / codeStep / dataStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Request is Optional<T>. The
  RequestView row type uses the same Optional wrapping.
- (Lesson 7) RouterTasks.java declares the EXECUTE Task<TaskResult> constant.
  All three specialists' definition().capability(TaskAcceptance.of(EXECUTE)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9866 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier label is "Runs out of the box" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour white, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels.
- The RoutingGuardrail runs INSIDE the workflow BEFORE the specialist is invoked —
  not inside the specialist's prompt and not after the specialist has produced output.
- The RoutingEvalScorer Consumer reacts to ClassificationDecided events and calls
  RoutingJudge via componentClient.forAgent(). It does NOT modify the workflow
  flow — the eval is out-of-band metadata.
- The guardrail step happens BEFORE any specialist is called. A blocked request
  never reaches any specialist — only the operator sees it in a "blocked" surface.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, use, use, marketing
  tone, competitor brand names.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local key-source reference written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
