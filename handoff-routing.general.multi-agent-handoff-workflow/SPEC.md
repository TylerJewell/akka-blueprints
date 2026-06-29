# SPEC — multi-agent-handoff-workflow

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Multi-Agent Handoff Workflow.
**One-line pitch:** A router agent classifies an incoming task request and hands execution off to a data-analyst, content-writer, or code-reviewer specialist that owns the result end-to-end, with admission control before every agent activation, a before-tool-call guardrail on specialist tools, and an inline eval scoring every routing decision.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *who* should own the task, then transfers the same task identity to a downstream specialist agent that produces the final output. The downstream agent is responsible for the entire result; the classifier does not narrate or synthesise. Three governance mechanisms are layered on top:

- A **before-agent-invocation guardrail** runs inside a Consumer before any agent is activated. It checks the incoming task against admission criteria (required fields present, content within acceptable scope, no prohibited topics) and blocks the request from entering the workflow when it fails. No agent call is made for a rejected request.
- A **before-tool-call guardrail** intercepts every tool invocation that a specialist agent attempts. It checks that the tool name, arguments, and calling context conform to allowed-tool policy (no external network calls outside approved domains, no file-system writes, no shell commands). A blocked tool call surfaces as a `TOOL_BLOCKED` terminal state.
- An **on-decision eval** fires every time the router agent emits a routing decision. A `RoutingJudge` agent grades the decision against the task payload on a 1–5 rubric. The score and rationale are written back to the task entity and surfaced in the UI.

The pattern is a fan-out-of-one: the workflow branches on the classifier's category, and only the chosen specialist is invoked. The other specialists see no traffic for that task.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live task list. Every task displays its domain chip, status pill, routing score, and (if resolved) the published result.
2. `TaskSimulator` (TimedAction) ticks every 30 s and inserts a new canned request from `sample-events/task-requests.jsonl` into `TaskQueue`.
3. For each new request: `AdmissionGuardrail` (Consumer) checks admission criteria; on pass, registers a `TaskEntity` and starts a `HandoffWorkflow`.
4. The workflow calls `RouterAgent`, gets a `RoutingDecision { domain, confidence, reason }`, and emits `RoutingDecided` on the task entity.
5. Branch on `domain`:
   - `DATA_ANALYSIS` → workflow calls `DataAnalyst` with the `EXECUTE` task and waits for the typed `TaskResult`.
   - `CONTENT_WRITING` → workflow calls `ContentWriter` with the same `EXECUTE` task.
   - `CODE_REVIEW` → workflow calls `CodeReviewer` with the same `EXECUTE` task.
   - `UNROUTABLE` → workflow emits `TaskRejected`; ends.
6. Each specialist's tool calls pass through the before-tool-call guardrail. If a tool call is blocked, `ToolCallBlocked` is emitted (terminal `TOOL_BLOCKED`) with the blocked tool and reason.
7. When the specialist returns a `TaskResult`, a final validation step checks that the result fields are non-empty and the result type matches the declared domain. On pass, `ResultPublished` is emitted (terminal `COMPLETED`). On fail, `ValidationFailed` is emitted (terminal `VALIDATION_FAILED`).
8. Independent of the workflow, `RoutingEvalScorer` (Consumer) listens for `RoutingDecided` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the task entity.
9. The user can click any task card and see the request payload, the routing reason, the routing score, the chosen specialist, the result (or blocked tool + reason), and the published output.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskSimulator` | `TimedAction` | Drips simulated task requests into `TaskQueue` every 30 s. | scheduler | `TaskQueue` |
| `TaskQueue` | `EventSourcedEntity` | Append-only audit log of every inbound task request (`InboundTaskReceived`). | `TaskSimulator`, `WorkflowEndpoint` | `AdmissionGuardrail` |
| `AdmissionGuardrail` | `Consumer` | Subscribes to `TaskQueue` events; checks admission; registers `TaskEntity`; starts a `HandoffWorkflow`. | `TaskQueue` events | `TaskEntity`, `HandoffWorkflow` |
| `RouterAgent` | `Agent` (typed, not autonomous) | Classifies an admitted request into `DATA_ANALYSIS` / `CONTENT_WRITING` / `CODE_REVIEW` / `UNROUTABLE` with confidence + reason. | invoked by `HandoffWorkflow` | returns `RoutingDecision` |
| `DataAnalyst` | `AutonomousAgent` | Owns the `EXECUTE` task for data-analysis work items. Returns typed `TaskResult`. | invoked by `HandoffWorkflow` | returns `TaskResult` |
| `ContentWriter` | `AutonomousAgent` | Owns the `EXECUTE` task for content-writing work items. Returns typed `TaskResult`. | invoked by `HandoffWorkflow` | returns `TaskResult` |
| `CodeReviewer` | `AutonomousAgent` | Owns the `EXECUTE` task for code-review work items. Returns typed `TaskResult`. | invoked by `HandoffWorkflow` | returns `TaskResult` |
| `RoutingJudge` | `Agent` (typed) | Grades a routing decision against the task payload. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RoutingEvalScorer` | returns `RoutingScore` |
| `HandoffWorkflow` | `Workflow` | Per-task orchestration: admit → route → branch → execute → validate → publish. | `AdmissionGuardrail` (start) | `TaskEntity` |
| `TaskEntity` | `EventSourcedEntity` | Per-task lifecycle. | `HandoffWorkflow`, `RoutingEvalScorer` | `TaskView` |
| `TaskView` | `View` | Read-model row per task. | `TaskEntity` events | `WorkflowEndpoint` |
| `RoutingEvalScorer` | `Consumer` | Subscribes to `TaskEntity` events; on `RoutingDecided` invokes `RoutingJudge` and writes `RoutingScored` back. | `TaskEntity` events | `TaskEntity` |
| `WorkflowEndpoint` | `HttpEndpoint` | `/api/tasks/*` — list, get, manual submit, SSE; `/api/metadata/*`. | — | `TaskView`, `TaskEntity`, `TaskQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record IncomingTask(
    String taskId,
    String requesterId,
    String title,
    String description,
    String preferredDomain,    // "data-analysis" | "content-writing" | "code-review" | "auto"
    Instant receivedAt
) {}

record AdmissionResult(
    boolean admitted,
    List<String> rejectionReasons   // empty when admitted
) {}

enum TaskDomain { DATA_ANALYSIS, CONTENT_WRITING, CODE_REVIEW, UNROUTABLE }

record RoutingDecision(
    TaskDomain domain,
    String confidence,             // "high" | "medium" | "low"
    String reason                  // one short sentence
) {}

enum ResultFormat { MARKDOWN_REPORT, STRUCTURED_JSON, INLINE_COMMENT, PLAIN_TEXT }

record TaskResult(
    String headline,
    String body,
    ResultFormat format,
    String specialistTag,          // "data-analyst" | "content-writer" | "code-reviewer"
    Instant completedAt
) {}

record ToolCallRecord(
    String toolName,
    String argsDigest,             // SHA-256 prefix of serialised args
    boolean allowed,
    Optional<String> blockReason,
    Instant calledAt
) {}

record RoutingScore(
    int score,                     // 1..5
    String rationale,
    Instant scoredAt
) {}

record Task(
    String taskId,
    IncomingTask incoming,
    Optional<AdmissionResult> admission,
    Optional<RoutingDecision> routing,
    Optional<TaskResult> result,
    Optional<ToolCallRecord> blockedTool,
    Optional<RoutingScore> routingScore,
    Optional<String> rejectionReason,
    TaskStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus {
    RECEIVED,
    ADMITTED,
    ROUTED,
    ROUTED_DATA_ANALYSIS,
    ROUTED_CONTENT_WRITING,
    ROUTED_CODE_REVIEW,
    EXECUTING,
    TOOL_BLOCKED,
    VALIDATION_FAILED,
    COMPLETED,
    REJECTED
}
```

Events on `TaskEntity`: `TaskReceived`, `TaskAdmitted`, `TaskRejectedAtAdmission`, `RoutingDecided`, `TaskRouted`, `ToolCallBlocked`, `ResultDrafted`, `ValidationFailed`, `ResultPublished`, `TaskRejected`, `RoutingScored`.

Events on `TaskQueue`: `InboundTaskReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/tasks` — list all tasks (newest-first), optional `?domain=DATA_ANALYSIS|CONTENT_WRITING|CODE_REVIEW|UNROUTABLE&status=…` filtered client-side.
- `GET /api/tasks/{id}` — one task.
- `POST /api/tasks` — manually submit a request (body `IncomingTask` minus `taskId` and `receivedAt`); server assigns both.
- `GET /api/tasks/sse` — Server-Sent Events for every task change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Multi-Agent Handoff Workflow</title>`.

The App UI tab is a three-pane layout: **left** is the task list (status pill + domain chip + routing-score chip), **centre** is the selected task's request + routing decision + score, **right** is the chosen specialist's result + tool-call record + published output (or tool-block reason + status when `TOOL_BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-invocation guardrail** (`AdmissionGuardrail` Consumer): checks every inbound task against admission criteria (required fields populated, description length reasonable, no prohibited-topic keywords, `preferredDomain` value is a known enum or `"auto"`). Blocking — a failed admission emits `TaskRejectedAtAdmission` without activating any agent.
- **G2 — before-tool-call guardrail** (enforced inside each specialist's tool-dispatch loop): checks every tool call attempted by a specialist against an allowed-tool registry (permitted tool names, argument shape, calling context). Blocking — a blocked tool call emits `ToolCallBlocked` and terminates the workflow in `TOOL_BLOCKED`.
- **E1 — on-decision eval** (`eval-event`, on the routing decision): `RoutingEvalScorer` (Consumer) listens for `RoutingDecided` events and calls `RoutingJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate.

## 9. Agent prompts

- `RouterAgent` → `prompts/router-agent.md`. Typed classifier; returns one of `DATA_ANALYSIS`, `CONTENT_WRITING`, `CODE_REVIEW`, `UNROUTABLE`; defaults to `UNROUTABLE` under ambiguity.
- `DataAnalyst` → `prompts/data-analyst.md`. Owns the `EXECUTE` task for data-analysis work items. Never invents data; routes outside its scope to `UNROUTABLE`.
- `ContentWriter` → `prompts/content-writer.md`. Owns the `EXECUTE` task for content-writing work items. Never plagiarises; never invents citations.
- `CodeReviewer` → `prompts/code-reviewer.md`. Owns the `EXECUTE` task for code-review work items. Never executes code; comments only on what is in the snippet.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a routing decision against a 1–5 rubric with one-sentence rationale.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a data-analysis task → admitted → routed `DATA_ANALYSIS` → executed by `DataAnalyst` → result published.
2. **J2** — Simulator drips a content-writing task → admitted → routed `CONTENT_WRITING` → executed by `ContentWriter` → result published.
3. **J3** — An unroutable task (empty description) routes as `UNROUTABLE` and lands in `REJECTED` without any specialist invocation.
4. **J4** — A task that fails admission (prohibited topic keyword) is blocked at the `AdmissionGuardrail` step; no agent is activated.
5. **J5** — A specialist that tries to call a disallowed tool is blocked by the before-tool-call guardrail; the task lands in `TOOL_BLOCKED`.
6. **J6** — Every routed task carries a `RoutingScore` (1–5) and rationale within ~10 s of the routing decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-agent-handoff-workflow demonstrating the handoff-routing × general cell.
Runs out of the box (in-process simulated inbound task stream; no real external task-management integration).
Maven group io.akka.samples. Maven artifact handoff-routing-general-multi-agent-handoff-workflow.
Java package io.akka.samples.multiagentworkflow. Akka 3.6.0. HTTP port 9535.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) RouterAgent — classifier. System prompt loaded from
  prompts/router-agent.md. Input: IncomingTask{taskId, requesterId, title, description,
  preferredDomain, receivedAt}. Output: RoutingDecision{domain: TaskDomain
  (DATA_ANALYSIS/CONTENT_WRITING/CODE_REVIEW/UNROUTABLE), confidence: "high"|"medium"|"low",
  reason: String}. Defaults to UNROUTABLE under uncertainty.

- 1 AutonomousAgent DataAnalyst — definition() with capability(TaskAcceptance.of(EXECUTE)
  .maxIterationsPerTask(4)). System prompt from prompts/data-analyst.md. Input:
  IncomingTask + RoutingDecision. Output: TaskResult{headline, body, format: ResultFormat,
  specialistTag = "data-analyst", completedAt}. Never invents data; scopes findings to
  what is in the task description.

- 1 AutonomousAgent ContentWriter — definition() with capability(TaskAcceptance.of(EXECUTE)
  .maxIterationsPerTask(4)). System prompt from prompts/content-writer.md. Same input
  shape; specialistTag = "content-writer".

- 1 AutonomousAgent CodeReviewer — definition() with capability(TaskAcceptance.of(EXECUTE)
  .maxIterationsPerTask(4)). System prompt from prompts/code-reviewer.md. Same
  input shape; specialistTag = "code-reviewer".

- 1 Agent (typed) RoutingJudge — judge. System prompt from prompts/routing-judge.md. Input:
  IncomingTask + RoutingDecision. Output: RoutingScore{score: int 1–5, rationale: String,
  scoredAt: Instant}.

- 1 Workflow HandoffWorkflow per taskId. Steps:
    routeStep -> branchStep -> {dataStep | contentStep | codeStep | rejectStep}
               -> validateStep -> publishStep
  routeStep calls componentClient.forAgent().inSession(taskId).method(RouterAgent::route)
    .invoke(incoming). On success emits RoutingDecided via TaskEntity.recordRouting.
  branchStep branches on RoutingDecision.domain:
    DATA_ANALYSIS -> proceed to dataStep (emits TaskRouted{DATA_ANALYSIS})
    CONTENT_WRITING -> proceed to contentStep (emits TaskRouted{CONTENT_WRITING})
    CODE_REVIEW -> proceed to codeStep (emits TaskRouted{CODE_REVIEW})
    UNROUTABLE -> rejectStep (emits TaskRejected; terminates).
  dataStep / contentStep / codeStep call forAutonomousAgent(<Specialist>.class, taskId)
    .runSingleTask(TaskDef.instructions(buildPrompt(incoming, routing))) returning a taskId,
    then forTask(taskId).result(WorkflowTasks.EXECUTE) to block on the typed TaskResult.
    Each specialist's tool calls are intercepted; any call to a tool not in the allowed-tool
    registry emits ToolCallBlocked (terminal TOOL_BLOCKED) and the step fails fast.
    On success emits ResultDrafted.
  validateStep checks that result.headline is non-empty and result.body length > 20 chars
    and result.specialistTag matches the expected specialist for this domain. On pass
    proceed to publishStep (emits ResultPublished, terminal COMPLETED). On fail emit
    ValidationFailed (terminal VALIDATION_FAILED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on routeStep and
    validateStep, stepTimeout(Duration.ofSeconds(60)) on dataStep, contentStep, codeStep,
    and publishStep. defaultStepRecovery(maxRetries(2).failoverTo(HandoffWorkflow::error)).

- 2 EventSourcedEntities:
    * TaskQueue — append-only audit log. Command receive(IncomingTask) emits
      InboundTaskReceived{incoming}. No mutable state beyond a counter; commands are
      idempotent on incoming.taskId.
    * TaskEntity (one per taskId) — full per-task lifecycle. State Task{taskId,
      incoming: IncomingTask, Optional<AdmissionResult> admission,
      Optional<RoutingDecision> routing, Optional<TaskResult> result,
      Optional<ToolCallRecord> blockedTool, Optional<RoutingScore> routingScore,
      Optional<String> rejectionReason, TaskStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. TaskStatus enum: RECEIVED, ADMITTED,
      ROUTED, ROUTED_DATA_ANALYSIS, ROUTED_CONTENT_WRITING, ROUTED_CODE_REVIEW,
      EXECUTING, TOOL_BLOCKED, VALIDATION_FAILED, COMPLETED, REJECTED.
      Events: TaskReceived, TaskAdmitted, TaskRejectedAtAdmission, RoutingDecided,
      TaskRouted, ToolCallBlocked, ResultDrafted, ValidationFailed, ResultPublished,
      TaskRejected, RoutingScored. Commands: registerIncoming, recordAdmission,
      rejectAtAdmission, recordRouting, recordTaskRouted, recordToolBlock, recordDraft,
      recordValidationFailure, publish, reject, recordRoutingScore, getTask.
      emptyState() returns Task.initial("") with no commandContext() reference.

- 2 Consumers:
    * AdmissionGuardrail subscribed to TaskQueue events; for each InboundTaskReceived
      applies admission checks (title non-empty, description length >= 10 chars and
      <= 4000 chars, preferredDomain is one of the valid values or "auto", no keyword
      from a prohibited-topics list), produces AdmissionResult with rejectionReasons,
      and calls TaskEntity.registerIncoming then recordAdmission (or rejectAtAdmission
      if not admitted) for the taskId; if admitted, starts a HandoffWorkflow
      with taskId as the workflow id.
    * RoutingEvalScorer subscribed to TaskEntity events; on RoutingDecided invokes
      RoutingJudge.score(incoming, decision) and calls TaskEntity.recordRoutingScore(
      taskId, score). On any other event type, no-op. Use componentClient — do NOT
      call the agent from a TimedAction.

- 1 View TaskView with row type TaskRow (mirrors Task; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). Table updater consumes TaskEntity events.
  ONE query getAllTasks SELECT * AS tasks FROM task_view. No WHERE domain or
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side in callers.

- 1 TimedAction TaskSimulator — every 30s, reads next line from
  src/main/resources/sample-events/task-requests.jsonl (loops at EOF) and calls
  TaskQueue.receive with a fresh taskId (UUID).

- 2 HttpEndpoints:
    * WorkflowEndpoint at /api with GET /tasks (list from TaskView.getAllTasks,
      filter client-side by ?domain and ?status query params), GET /tasks/{id},
      POST /tasks (body IncomingTask minus taskId/receivedAt — server assigns),
      GET /tasks/sse (serverSentEventsForView over getAllTasks), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- WorkflowTasks.java declaring the task constants: EXECUTE (resultConformsTo TaskResult.class,
  description "Execute the assigned work item end-to-end and return a typed TaskResult").
- Domain records IncomingTask, AdmissionResult, RoutingDecision, TaskResult,
  ToolCallRecord, RoutingScore, and the Task entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9535 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/task-requests.jsonl with 9 canned lines (3 data-analysis-
  flavoured, 2 content-writing-flavoured, 2 code-review-flavoured, 1 unroutable one-liner,
  1 designed to trip the before-tool-call guardrail by requesting an external web fetch).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls: G1 guardrail before-agent-invocation,
  G2 guardrail before-tool-call, E1 eval-event on-decision-eval. Matching
  simplified_view list. No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = task-routing,
  data.data_classes.pii = false, decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false (the system publishes without HITL by default — only
  tool-blocked tasks wait for review), failure.failure_modes including "wrong-domain-routing",
  "out-of-scope-tool-use", "result-hallucination", "admission-bypass"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/router-agent.md, prompts/data-analyst.md, prompts/content-writer.md,
  prompts/code-reviewer.md, prompts/routing-judge.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Multi-Agent Handoff Workflow", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = task list with domain chip + status pill + routing-score chip; centre = request
  detail + routing block; right = specialist result + tool-call record + published output
  or tool-block reason). Browser title exactly:
  <title>Akka Sample: Multi-Agent Handoff Workflow</title>. No subtitle on the Overview tab.

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
  pseudo-randomly per call (seeded by taskId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    router-agent.json — 12 RoutingDecision entries spanning DATA_ANALYSIS (trend
      analysis, metrics summary, data-quality check), CONTENT_WRITING (blog post,
      email draft, product description), CODE_REVIEW (pull-request review, function
      audit, test coverage check), and UNROUTABLE (empty description, off-topic,
      ambiguous one-liner). Confidence + a one-sentence reason on each.
    data-analyst.json — 8 TaskResult entries: 5 with format MARKDOWN_REPORT or
      STRUCTURED_JSON (scoped findings with caveats), 1 with format PLAIN_TEXT,
      1 with ESCALATED-equivalent body (requests missing data), 1 designed to trip
      the before-tool-call guardrail by attempting an external web fetch.
    content-writer.json — 8 TaskResult entries: 5 with format MARKDOWN_REPORT or
      PLAIN_TEXT, 1 with ARTICLE_LINKED-equivalent note, 1 with ESCALATED-equivalent
      (requests clarification), 1 designed to trip the before-tool-call guardrail.
    code-reviewer.json — 8 TaskResult entries: 5 with INLINE_COMMENT or
      MARKDOWN_REPORT, 1 with scope-limited review, 1 with ESCALATED-equivalent,
      1 designed to trip the before-tool-call guardrail (attempts shell command).
    routing-judge.json — 10 RoutingScore entries, score 1–5, one-sentence
      rationale matching the rubric (domain-correctness / confidence-calibration /
      reason-quality).
- A MockModelProvider.seedFor(taskId) helper makes per-task selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  DataAnalyst, ContentWriter, and CodeReviewer all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): routeStep 20s,
  validateStep 20s, dataStep / contentStep / codeStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Task is Optional<T>. The
  TaskView row type uses the same Optional wrapping.
- (Lesson 7) WorkflowTasks.java declares the EXECUTE Task<TaskResult> constant.
  All three specialists' definition().capability(TaskAcceptance.of(EXECUTE)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9535 in application.conf; not 9000.
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
- The AdmissionGuardrail runs INSIDE a Consumer before any LLM call — not inside
  an Agent's prompt and not after the LLM has seen the raw payload.
- The RoutingEvalScorer Consumer reacts to RoutingDecided events and calls
  RoutingJudge via componentClient.forAgent(). It does NOT modify the workflow
  flow — the eval is out-of-band metadata.
- The before-tool-call guardrail intercepts BEFORE the tool executes. A blocked
  tool call never runs; the task terminates in TOOL_BLOCKED with the tool name
  and reason surfaced for operator review.
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
