# SPEC — aws-ops-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AwsOpsAssistant.
**One-line pitch:** A user types a natural-language AWS operations request; one AI agent translates it into MCP-exposed AWS API calls, gates every mutating action behind a human confirmation step, and returns a structured operation report — COMPLETED / BLOCKED / HALTED — with one action entry per AWS call made.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `AwsOpsAgent` (AutonomousAgent) handles all AWS API orchestration; the surrounding components only prepare its permissions context, gate mutating calls, and audit its output. Three governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs before any AWS API call the agent attempts — so disallowed resource types (outside the configured allow-list) are blocked before any network call leaves the process.
- An **application-level HITL (human-in-the-loop)** pauses the agent before each mutating call (create, update, delete, attach, detach) and presents a confirmation card in the UI. The agent resumes only after the user approves or declines each action.
- An **operator halt** control lets an operator or automated circuit-breaker freeze the running agent at any point, transitioning the request to HALTED and preventing any further AWS API calls. This is the wide-blast-radius safety valve: a single running agent could touch many resources across many services in seconds.

The blueprint shows that a single-agent architecture can carry rigorous operational governance — three independent controls sit on different sides of the one decision-making LLM.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types an operations request into the **Request** textarea (or picks one of three seeded examples — list all EC2 instances in us-east-1, report S3 bucket storage sizes, resize an EC2 instance).
2. The user selects a **scope** from a dropdown (read-only / mutating / mixed) and fills in an optional **Context** field (e.g., environment tag, account alias).
3. The user clicks **Submit request**. The UI POSTs to `/api/ops-requests` and receives an `opsRequestId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `PLANNING` — the agent has parsed the request and produced a plan of AWS API calls.
5. For each mutating call the agent plans, the workflow pauses in `AWAITING_CONFIRMATION` and the UI shows a confirmation card: the AWS API name, resource ARN or prefix, and a short rationale. The user clicks **Approve** or **Decline**.
6. On approval, the agent executes the call. On decline, the agent records a SKIPPED entry and continues to the next planned action. When all actions are resolved, the card transitions to `COMPLETED`.
7. An operator can click the red **Halt** button at any point — the request transitions immediately to `HALTED`; the agent receives a halt signal and stops issuing calls.
8. The final report card shows a top-level status badge (COMPLETED / BLOCKED / HALTED), a summary paragraph, and a per-action table (service, action, resource, outcome, duration).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OpsEndpoint` | `HttpEndpoint` | `/api/ops-requests/*` — submit, list, get, confirm, halt, SSE; serves `/api/metadata/*`. | — | `OpsRequestEntity`, `OpsView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `OpsRequestEntity` | `EventSourcedEntity` | Per-request lifecycle: submitted → planning → awaiting-confirmation → executing → completed/halted. Source of truth. | `OpsEndpoint`, `ConfirmationConsumer`, `OpsWorkflow` | `OpsView` |
| `ConfirmationConsumer` | `Consumer` | Subscribes to `ConfirmationRequested` events; routes the pending action to the UI confirmation queue via `OpsRequestEntity.recordConfirmationDelivered`. | `OpsRequestEntity` events | `OpsRequestEntity` |
| `OpsWorkflow` | `Workflow` | One workflow per request. Steps: `planStep` → `confirmStep` (loops per mutating action) → `executeStep` → `reportStep`. | started by `OpsEndpoint` on submit | `AwsOpsAgent`, `OpsRequestEntity` |
| `AwsOpsAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the operations request plus context; calls MCP-exposed AWS tools one at a time; pauses for HITL confirmation before each mutating call; returns `OperationReport`. | invoked by `OpsWorkflow` | returns report |
| `ActionGuardrail` | — | `before-tool-call` hook registered on `AwsOpsAgent`. Checks every planned tool call against the allowed-resource-types list before execution. | `AwsOpsAgent` | `AwsOpsAgent` (pass or reject) |
| `OpsView` | `View` | Read model: one row per request for the UI. | `OpsRequestEntity` events | `OpsEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record AwsAction(
    String actionId,
    String awsService,       // e.g. "EC2", "S3", "Lambda"
    String apiCall,          // e.g. "DescribeInstances", "PutObject", "UpdateFunctionCode"
    String resourceArn,      // ARN or prefix; "*" for list-style reads
    ActionKind kind,         // READ or MUTATING
    String rationale         // one-sentence agent justification
) {}
enum ActionKind { READ, MUTATING }

record ConfirmationRequest(
    String actionId,
    AwsAction action,
    Instant requestedAt
) {}

record ConfirmationDecision(
    String actionId,
    ConfirmationOutcome outcome,
    String decidedBy,
    Instant decidedAt
) {}
enum ConfirmationOutcome { APPROVED, DECLINED }

record ActionResult(
    String actionId,
    AwsAction action,
    ActionOutcome outcome,   // COMPLETED, SKIPPED, BLOCKED, FAILED
    String responseSnippet,  // truncated API response (≤ 512 chars)
    long durationMs
) {}
enum ActionOutcome { COMPLETED, SKIPPED, BLOCKED, FAILED }

record OpsRequest(
    String opsRequestId,
    String requestText,
    String context,          // optional environment tag / account alias
    RequestScope scope,      // READ_ONLY, MUTATING, MIXED
    String submittedBy,
    Instant submittedAt
) {}
enum RequestScope { READ_ONLY, MUTATING, MIXED }

record OperationReport(
    ReportStatus status,
    String summary,
    List<ActionResult> actions,
    Instant completedAt
) {}
enum ReportStatus { COMPLETED, BLOCKED, HALTED }

record OpsRequestState(
    String opsRequestId,
    Optional<OpsRequest> request,
    List<ConfirmationRequest> pendingConfirmations,
    List<ConfirmationDecision> decisions,
    Optional<OperationReport> report,
    OpsRequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum OpsRequestStatus {
    SUBMITTED, PLANNING, AWAITING_CONFIRMATION, EXECUTING, COMPLETED, HALTED, FAILED
}
```

Events on `OpsRequestEntity`: `RequestSubmitted`, `PlanningStarted`, `ConfirmationRequested`, `ConfirmationDelivered`, `ConfirmationReceived`, `ExecutionStarted`, `ReportRecorded`, `RequestHalted`, `RequestFailed`.

Every nullable lifecycle field on `OpsRequestState` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/ops-requests` — body `{ requestText, context, scope, submittedBy }` → `{ opsRequestId }`.
- `GET /api/ops-requests` — list all requests, newest-first.
- `GET /api/ops-requests/{id}` — one request.
- `POST /api/ops-requests/{id}/confirm` — body `{ actionId, outcome, decidedBy }` → `204`.
- `POST /api/ops-requests/{id}/halt` — body `{ haltedBy }` → `204`.
- `GET /api/ops-requests/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: AWS Ops Assistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted requests (status pill + report-status badge + age) and a right pane with the selected request's detail — request text, planning summary, per-action confirmation cards (for pending HITL actions), action results table, and the final operation report.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`): runs before every MCP tool call `AwsOpsAgent` attempts. Reads the target `awsService` and `resourceArn` from the planned action and checks them against an `allowed-resource-types` list configured in `application.conf`. If the service or resource prefix is outside the allow-list, returns a structured `blocked-resource` rejection to the agent loop; the agent records a BLOCKED `ActionResult` and moves to the next planned action without making the API call.
- **H1 — application HITL** (`hitl`, `application`): runs before each mutating action (kind == MUTATING). The workflow pauses in `confirmStep`, emits `ConfirmationRequested`, and blocks until a `ConfirmationReceived` event arrives (user clicks Approve or Decline in the UI). Approved actions advance to execution; declined actions are recorded as SKIPPED. The confirmation window is bounded by a 10-minute step timeout; expiry records the action as SKIPPED automatically.
- **X1 — operator halt** (`halt`, `operator-regulator-stop`): exposed as `POST /api/ops-requests/{id}/halt`. Emits `RequestHalted` on the entity; the workflow monitors for this event between every action and stops issuing further calls when it sees it. The halt is immediate — any action currently in-flight completes its current API call (one atomic operation) before the workflow checks the halt flag. No further actions are issued after the flag is observed.

## 9. Agent prompts

- `AwsOpsAgent` → `prompts/aws-ops-agent.md`. The single decision-making LLM. System prompt instructs it to parse the operations request, produce an ordered plan of AWS API calls, and execute them one at a time — pausing for HITL confirmation before each mutating call and recording the API response as an `ActionResult`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the read-only EC2 list request; within 30 s the report appears with one `ActionResult` per describe call and no confirmation prompts.
2. **J2** — User submits the EC2 resize request; the UI shows a confirmation card for `ModifyInstanceAttribute`; user approves; the action executes; the report lands with status COMPLETED.
3. **J3** — Operator clicks Halt during a multi-action request; the request transitions to HALTED; subsequent planned actions are not executed.
4. **J4** — A mutating call targeting `DynamoDB` (outside the allow-list) is rejected by the guardrail before any API call; the report shows a BLOCKED `ActionResult` for that action.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named aws-ops-assistant demonstrating the single-agent × dev-code cell.
Runs out of the box (no external AWS account required — AWS MCP surface simulated in-process).
Maven group io.akka.samples. Maven artifact single-agent-dev-code-aws-ops-assistant.
Java package io.akka.samples.awsassistantmcp. Akka 3.6.0. HTTP port 9276.

Components to wire (exactly):

- 1 AutonomousAgent AwsOpsAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/aws-ops-agent.md>) and
  .capability(TaskAcceptance.of(OPS_TASKS.EXECUTE_OPS_REQUEST).maxIterationsPerTask(12)).
  The task receives the request text and context in the task instructions; the agent calls
  MCP-exposed AWS tool stubs (DescribeInstances, ListBuckets, ModifyInstanceAttribute, etc.)
  one at a time. Output: OperationReport{status: ReportStatus, summary: String,
  actions: List<ActionResult>, completedAt: Instant}. The agent is configured with a
  before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent records a BLOCKED
  ActionResult and continues planning.

- 1 Workflow OpsWorkflow per opsRequestId with four logical steps:
  * planStep — emits PlanningStarted, then calls AwsOpsAgent with the request text and
    context. WorkflowSettings.stepTimeout 15s (planning is fast; no AWS calls yet).
  * confirmStep — loops once per pending ConfirmationRequest: emits ConfirmationRequested,
    then polls OpsRequestEntity.getRequest every 2s waiting for ConfirmationReceived.
    WorkflowSettings.stepTimeout 600s (10-minute human window). On timeout or DECLINED,
    records SKIPPED ActionResult.
  * executeStep — for each APPROVED action calls the corresponding AWS MCP stub, records
    ActionResult, continues to next. WorkflowSettings.stepTimeout 120s with
    defaultStepRecovery maxRetries(1).failoverTo(OpsWorkflow::error).
  * reportStep — assembles OperationReport from all ActionResult entries; calls
    OpsRequestEntity.recordReport(report). WorkflowSettings.stepTimeout 10s.
    error step emits RequestFailed and transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). Between each step the workflow checks
  OpsRequestEntity.getRequest for RequestHalted before proceeding (halt support).

- 1 EventSourcedEntity OpsRequestEntity (one per opsRequestId). State OpsRequestState{
  opsRequestId: String, request: Optional<OpsRequest>,
  pendingConfirmations: List<ConfirmationRequest>, decisions: List<ConfirmationDecision>,
  report: Optional<OperationReport>, status: OpsRequestStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. OpsRequestStatus enum: SUBMITTED, PLANNING,
  AWAITING_CONFIRMATION, EXECUTING, COMPLETED, HALTED, FAILED. Events: RequestSubmitted{request},
  PlanningStarted{}, ConfirmationRequested{confirmationRequest}, ConfirmationDelivered{actionId},
  ConfirmationReceived{decision}, ExecutionStarted{}, ReportRecorded{report},
  RequestHalted{haltedBy, haltedAt}, RequestFailed{reason}. Commands: submit, startPlanning,
  requestConfirmation, recordConfirmationDelivered, receiveConfirmation, startExecution,
  recordReport, halt, fail, getRequest. emptyState() returns OpsRequestState.initial("") with
  no commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 Consumer ConfirmationConsumer subscribed to OpsRequestEntity events; on
  ConfirmationRequested calls OpsRequestEntity.recordConfirmationDelivered(actionId) so the
  workflow knows the UI has received the prompt. This prevents the workflow from timing out
  before the UI even renders the confirmation card.

- 1 View OpsView with row type OpsRequestRow (mirrors OpsRequestState minus
  pendingConfirmations detail — the row carries a count and the most-recent pending actionId).
  Table updater consumes OpsRequestEntity events. ONE query getAllRequests:
  SELECT * AS requests FROM ops_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * OpsEndpoint at /api with:
    POST /ops-requests (body {requestText, context, scope, submittedBy}; mints opsRequestId;
      calls OpsRequestEntity.submit; starts OpsWorkflow; returns {opsRequestId}),
    GET /ops-requests (list from getAllRequests, sorted newest-first),
    GET /ops-requests/{id} (one row),
    POST /ops-requests/{id}/confirm (body {actionId, outcome, decidedBy}; calls
      OpsRequestEntity.receiveConfirmation; returns 204),
    POST /ops-requests/{id}/halt (body {haltedBy}; calls OpsRequestEntity.halt; returns 204),
    GET /ops-requests/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- OpsTasks.java declaring one Task<R> constant: EXECUTE_OPS_REQUEST = Task.name("Execute
  AWS operations request").description("Parse the natural-language request, call AWS APIs
  one at a time via MCP tools, pause for human confirmation before mutating calls, and
  return an OperationReport").resultConformsTo(OperationReport.class). DO NOT skip this —
  the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records AwsAction, ActionKind, ConfirmationRequest, ConfirmationDecision,
  ConfirmationOutcome, ActionResult, ActionOutcome, OpsRequest, RequestScope,
  OperationReport, ReportStatus, OpsRequestState, OpsRequestStatus.

- ActionGuardrail.java implementing the before-tool-call hook. Reads the target awsService
  and resourceArn from the planned tool call, checks them against the
  allowed-resource-types list in application.conf (akka.javasdk.app.allowed-aws-services),
  and either passes the call through or returns Guardrail.reject("blocked-resource: " +
  awsService) to record a BLOCKED ActionResult. On rejection the agent loop does NOT retry
  the same call — it moves to the next planned action.

- AwsMcpStubs.java — in-process simulation of the MCP AWS tool surface. Implements stubs
  for: EC2.DescribeInstances, EC2.ModifyInstanceAttribute, S3.ListBuckets,
  S3.GetBucketLocation, Lambda.ListFunctions, Lambda.UpdateFunctionCode,
  IAM.ListRoles, CloudWatch.GetMetricStatistics. Each stub returns a realistic but
  synthetic JSON snippet (≤ 512 chars). Mutating stubs (Modify, Update, Put) accept
  any resource ARN and return a synthetic 200-OK with a fake request ID. READ stubs
  return lists of 2–4 synthetic resources. AwsMcpStubs is registered as the MCP tool
  provider in AwsOpsAgent.definition().

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9276 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also:
  akka.javasdk.app.allowed-aws-services = ["EC2", "S3", "Lambda", "IAM", "CloudWatch"]
  (the guardrail's allow-list). The AwsOpsAgent.definition() binds the configured provider
  via the per-agent or default model-provider pattern from the akka-context docs.

- src/main/resources/sample-events/operation-scripts.jsonl with 3 seeded request scripts:
  a read-only EC2 inventory (requestText: "List all EC2 instances in us-east-1 with their
  instance type and state", scope: READ_ONLY), an S3 storage report (requestText: "Report
  total storage size per S3 bucket and flag any buckets over 100 GB", scope: READ_ONLY),
  and an EC2 resize (requestText: "Resize instance i-0abc123 from t3.medium to t3.large",
  scope: MUTATING).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H1, X1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false,
  decisions.authority_level = execute (the agent acts, not just recommends — with HITL
  gating mutating actions), oversight.human_in_loop = true (every mutating call requires
  confirmation), failure.failure_modes including "unintended-resource-deletion",
  "scope-creep-across-accounts", "confirmation-fatigue-bypass", "halt-lag-on-wide-blast";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/aws-ops-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: AWS Ops Assistant (MCP)",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of request cards; right = selected-request detail with request text, planning
  summary, confirmation cards for pending HITL actions, action results table, and final
  operation report). Browser title exactly: <title>Akka Sample: AWS Ops Assistant</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(opsRequestId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    execute-aws-operations-request.json — 6 OperationReport entries covering COMPLETED and
      HALTED status values. Each entry has a summary paragraph and an actions list with 2–5
      ActionResult entries covering READ, MUTATING APPROVED, and MUTATING SKIPPED outcomes.
      Each ActionResult has a non-empty responseSnippet (a realistic AWS-style JSON fragment)
      and a durationMs between 80 and 2200. Plus 1 entry containing a BLOCKED ActionResult
      (DynamoDB targeted — outside the allow-list) to exercise the guardrail path. Include
      1 HALTED entry with partial actions to exercise J3.
- A MockModelProvider.seedFor(opsRequestId) helper makes per-request selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. AwsOpsAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion OpsTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (planStep 15s, confirmStep 600s,
  executeStep 120s, reportStep 10s, error 10s).
- Lesson 6: every nullable lifecycle field on OpsRequestState is Optional<T>.
- Lesson 7: OpsTasks.java with EXECUTE_OPS_REQUEST = Task.name(...).description(...)
  .resultConformsTo(OperationReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9276 declared explicitly in application.conf's
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
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: there is exactly ONE AutonomousAgent (AwsOpsAgent). The HITL
  confirmation is surfaced through the workflow and entity, not through a second agent.
- The operator halt is wired as a command on OpsRequestEntity and a workflow halt-check
  between steps — not as a second agent or an external service call.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as a post-hoc filter on the returned report.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
