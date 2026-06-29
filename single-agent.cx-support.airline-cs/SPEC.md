# SPEC — airline-cs

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AirlineCS.
**One-line pitch:** A customer types a service request (change seat, rebook flight, file complaint); one ReAct agent reasons over a set of booking tools until the task is resolved or declined, with a human-in-the-loop confirmation gate before any destructive reservation write executes.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `CustomerServiceAgent` (AutonomousAgent) carries the entire decision loop; the surrounding components prepare its input, govern its tool calls, and surface the outcome. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw customer message and the agent invocation — so the model never sees frequent-flyer numbers, passport-like tokens, or raw email addresses.
- A **before-tool-call guardrail** intercepts every call to `modifyReservation` and `cancelReservation` and asserts a valid confirmation token is present. A call without a token is blocked; the guardrail returns a structured rejection instructing the agent to first request customer confirmation via the `requestConfirmation` tool.
- A **human-in-the-loop confirmation gate** (application-level HITL) pauses the workflow when `ConfirmationRequested` is emitted. The workflow resumes only when the customer submits a `confirm` or `cancel` token through the API. This is the application-gated HITL flavor: the workflow itself is the gate, not an external review queue.

The blueprint shows that a ReAct agent with tool-level governance is not equivalent to an ungoverned loop — the guardrail and HITL gate together ensure no destructive write ever executes without explicit customer consent.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a customer message into the **Request** textarea (or picks one of three seeded examples — a seat upgrade request, a flight rebook request, a complaint about a missed connection).
2. The user fills in a **Booking reference** (e.g., `ABC123`) or leaves it blank for complaint-only flows. Clicks **Submit request**. The UI POSTs to `/api/requests` and receives a `requestId`.
3. The card appears in the live list in `RECEIVED` state. Within ~1 s it transitions to `SANITIZED` — the agent now has the redacted message.
4. The agent begins its ReAct loop. The card transitions to `PROCESSING`. Tool calls appear as a live log in the card detail.
5. If the agent decides to modify or cancel a reservation, the card transitions to `AWAITING_CONFIRMATION`. The right pane shows the proposed change and a **Confirm** / **Cancel** button pair. The workflow is paused.
6. The customer clicks **Confirm**. The UI POSTs `{ action: "confirm" }` to `/api/requests/{id}/confirmation`. The workflow resumes; the agent receives the confirmation token and re-invokes the tool. The card transitions back to `PROCESSING`.
7. Within ~30 s of initial submission, the card reaches `COMPLETED`. The right pane shows the `ServiceOutcome`: resolved action, summary, any case numbers, and the final tool call log.
8. If the customer clicks **Cancel**, the workflow receives `{ action: "cancel" }` and the agent stops the modification path, recording `CANCELLED` on the outcome.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ServiceEndpoint` | `HttpEndpoint` | `/api/requests/*` — submit, list, get, SSE, confirmation response; serves `/api/metadata/*`. | — | `RequestEntity`, `RequestView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `RequestEntity` | `EventSourcedEntity` | Per-request lifecycle: received → sanitized → processing → awaiting-confirmation → completed/cancelled/failed. Source of truth. | `ServiceEndpoint`, `MessageSanitizer`, `ServiceWorkflow` | `RequestView` |
| `MessageSanitizer` | `Consumer` | Subscribes to `RequestReceived` events; redacts PII from the customer message; calls `RequestEntity.attachSanitized`. | `RequestEntity` events | `RequestEntity` |
| `ServiceWorkflow` | `Workflow` | One workflow per request. Steps: `awaitSanitizedStep` → `serveStep` → `hitlStep` (conditional) → `completeStep`. | started by `MessageSanitizer` | `CustomerServiceAgent`, `RequestEntity` |
| `CustomerServiceAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the sanitized customer message and booking context; reasons over tools in a ReAct loop; returns `ServiceOutcome`. | invoked by `ServiceWorkflow` | returns outcome |
| `ModificationGuardrail` | supporting class | Registered as `before-tool-call` on `CustomerServiceAgent`. Blocks `modifyReservation` and `cancelReservation` calls that lack a valid confirmation token. | agent tool calls | pass-through or reject |
| `BookingService` | supporting class | In-process booking record store. Exposes `searchBooking(ref)`, `modifyReservation(ref, change, token)`, `cancelReservation(ref, token)`, `fileComplaint(ref, category, description)`. Used by the agent's tool implementations. | `CustomerServiceAgent` tool calls | in-memory records |
| `RequestView` | `View` | Read model: one row per request for the UI. | `RequestEntity` events | `ServiceEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record BookingChange(
    String changeType,     // "SEAT", "DATE", "FLIGHT", "CANCEL"
    String newValue,       // e.g. "14A", "2026-07-15", "AA456"
    String reason
) {}

record ServiceRequest(
    String requestId,
    String bookingRef,     // may be empty for complaint-only flows
    String rawMessage,
    String submittedBy,
    Instant receivedAt
) {}

record SanitizedMessage(
    String redactedMessage,
    List<String> piiCategoriesFound
) {}

record ToolCall(
    String toolName,
    String arguments,
    String result,
    Instant calledAt
) {}

record ConfirmationRequest(
    String proposedChange,   // human-readable description of what will be modified
    String confirmationToken // opaque token the guardrail validates
) {}

record ServiceOutcome(
    OutcomeStatus status,
    String summary,
    List<ToolCall> toolCallLog,
    String caseNumber,       // non-null when a complaint was filed
    Instant completedAt
) {}
enum OutcomeStatus { RESOLVED, CANCELLED, FAILED, NEEDS_FOLLOWUP }

record ServiceRequestState(
    String requestId,
    Optional<ServiceRequest> request,
    Optional<SanitizedMessage> sanitized,
    Optional<ConfirmationRequest> pendingConfirmation,
    Optional<ServiceOutcome> outcome,
    RequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus {
    RECEIVED, SANITIZED, PROCESSING, AWAITING_CONFIRMATION, COMPLETED, CANCELLED, FAILED
}
```

Events on `RequestEntity`: `RequestReceived`, `MessageSanitized`, `ProcessingStarted`, `ConfirmationRequested`, `ConfirmationReceived`, `RequestCompleted`, `RequestCancelled`, `RequestFailed`.

Every nullable lifecycle field on the `ServiceRequestState` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/requests` — body `{ bookingRef, rawMessage, submittedBy }` → `{ requestId }`.
- `GET /api/requests` — list all requests, newest-first.
- `GET /api/requests/{id}` — one request.
- `POST /api/requests/{id}/confirmation` — body `{ action: "confirm" | "cancel" }` → `{ requestId, status }`.
- `GET /api/requests/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: AirlineCS</title>`.

The App UI tab is a two-column layout: a left rail with the submission form and live list of requests (status pill + outcome badge + age) and a right pane with the selected request's detail — sanitized message preview, tool call log, pending confirmation panel (when status is `AWAITING_CONFIRMATION`), outcome summary, and case-number badge if a complaint was filed.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MessageSanitizer` Consumer): redacts emails, frequent-flyer numbers, passport-like tokens (alphanumeric 6–9 char), phone numbers, and person names from the raw customer message before any LLM call. Records which categories were found.
- **G1 — before-tool-call guardrail**: runs on every tool call by `CustomerServiceAgent`. For `modifyReservation` and `cancelReservation`, asserts a `confirmationToken` field is present in the arguments and matches the token issued in the most recent `ConfirmationRequested` event on the entity. A call without a valid token is rejected with a structured error that instructs the agent to call `requestConfirmation` first.
- **H1 — application HITL gate**: when the agent calls `requestConfirmation`, the workflow transitions `RequestEntity` to `AWAITING_CONFIRMATION` and pauses. The workflow remains paused until `ServiceEndpoint` receives a `POST /api/requests/{id}/confirmation` from the customer. On `confirm`, the workflow emits `ConfirmationReceived` (with the issued token) and resumes. On `cancel`, the workflow calls `RequestEntity.cancel` and records `CANCELLED`.

## 9. Agent prompts

- `CustomerServiceAgent` → `prompts/customer-service-agent.md`. The single decision-making LLM. System prompt instructs it to reason step-by-step, use the available tools in order (search before modify, request confirmation before any destructive write), and return a `ServiceOutcome` when the task is complete.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer asks to change seat on booking `ABC123`; agent searches, proposes change, HITL fires, customer confirms, seat is modified, outcome is `RESOLVED`.
2. **J2** — Agent attempts `modifyReservation` without a confirmation token; guardrail blocks it; agent calls `requestConfirmation`; customer confirms; second `modifyReservation` with token succeeds.
3. **J3** — Customer message contains `ff-number: FFP-1234567` and `email: jane@example.com`; LLM call log shows only redacted tokens; entity's `request.rawMessage` retains the raw form.
4. **J4** — Customer files a complaint about a missed connection; agent calls `fileComplaint`; outcome carries a non-null `caseNumber`; no HITL gate fires (complaints are non-destructive).

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named airline-cs demonstrating the single-agent × cx-support cell. Runs out of
the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-airline-cs. Java package io.akka.samples.customerservicereactagent.
Akka 3.6.0. HTTP port 9640.

Components to wire (exactly):

- 1 AutonomousAgent CustomerServiceAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/customer-service-agent.md>) and
  .capability(TaskAcceptance.of(HANDLE_SERVICE_REQUEST).maxIterationsPerTask(8)).
  The task receives the sanitized customer message and booking context as its instruction text.
  There is NO task attachment — the message is short enough for inline text (unlike a full
  document). Output: ServiceOutcome{status: OutcomeStatus, summary: String,
  toolCallLog: List<ToolCall>, caseNumber: String, completedAt: Instant}.
  The agent is configured with a before-tool-call guardrail (G1) registered via the
  agent's guardrail-configuration block.

- 1 Workflow ServiceWorkflow per requestId with four steps:
  * awaitSanitizedStep — polls RequestEntity.getRequest every 1s; on
    request.sanitized().isPresent() advances to serveStep.
    WorkflowSettings.stepTimeout 15s.
  * serveStep — emits ProcessingStarted, then calls
    componentClient.forAutonomousAgent(CustomerServiceAgent.class, "agent-" + requestId)
    .runSingleTask(TaskDef.instructions(formatRequest(request))) — returns taskId, then
    forTask(taskId).result(HANDLE_SERVICE_REQUEST) to fetch the ServiceOutcome. If the
    agent emits a ConfirmationRequested tool call mid-loop, the workflow transitions the
    entity to AWAITING_CONFIRMATION and suspends via hitlStep. WorkflowSettings.stepTimeout
    120s with defaultStepRecovery maxRetries(2).failoverTo(ServiceWorkflow::error).
  * hitlStep — the workflow is paused. A resume signal arrives when ServiceEndpoint receives
    POST /api/requests/{id}/confirmation. On action=confirm the workflow emits
    ConfirmationReceived (carrying the issued token) and transitions back into the agent's
    task loop (returning the token as a signal to the running task). On action=cancel, calls
    RequestEntity.cancel. WorkflowSettings.stepTimeout 3600s (customer may take time).
  * completeStep — calls RequestEntity.complete(outcome). WorkflowSettings.stepTimeout 5s.
    error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity RequestEntity (one per requestId). State ServiceRequestState{
  requestId: String, request: Optional<ServiceRequest>,
  sanitized: Optional<SanitizedMessage>,
  pendingConfirmation: Optional<ConfirmationRequest>,
  outcome: Optional<ServiceOutcome>, status: RequestStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. RequestStatus enum: RECEIVED, SANITIZED, PROCESSING,
  AWAITING_CONFIRMATION, COMPLETED, CANCELLED, FAILED. Events: RequestReceived{request},
  MessageSanitized{sanitized}, ProcessingStarted{}, ConfirmationRequested{confirmation},
  ConfirmationReceived{token: String}, RequestCompleted{outcome}, RequestCancelled{},
  RequestFailed{reason}. Commands: receive, attachSanitized, markProcessing,
  requestConfirmation, receiveConfirmation, complete, cancel, fail, getRequest.
  emptyState() returns ServiceRequestState.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer MessageSanitizer subscribed to RequestEntity events; on RequestReceived runs a
  regex+heuristic redaction pipeline (emails, frequent-flyer-number pattern FFP-\d{7},
  passport-like alphanumeric 6–9 char, phone numbers, person-name heuristic) over rawMessage,
  builds SanitizedMessage, then calls RequestEntity.attachSanitized(sanitized). After
  attachSanitized lands, the same Consumer starts a ServiceWorkflow with id =
  "service-" + requestId.

- 1 View RequestView with row type RequestRow (mirrors ServiceRequestState minus
  request.rawMessage). ONE query getAllRequests: SELECT * AS requests FROM request_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * ServiceEndpoint at /api with POST /requests (body {bookingRef, rawMessage, submittedBy};
    mints requestId; calls RequestEntity.receive; returns {requestId}), GET /requests (list
    from getAllRequests, sorted newest-first), GET /requests/{id} (one row), POST
    /requests/{id}/confirmation (body {action: "confirm"|"cancel"}; resumes or cancels the
    hitlStep in ServiceWorkflow; returns {requestId, status}), GET /requests/sse (SSE
    forwarded from view stream-updates), and three /api/metadata/* endpoints serving
    YAML/MD from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ServiceTasks.java declaring one Task<R> constant: HANDLE_SERVICE_REQUEST =
  Task.name("Handle service request").description("Reason over the customer's request
  using available booking tools and return a ServiceOutcome").resultConformsTo(
  ServiceOutcome.class). DO NOT skip this — the AutonomousAgent requires its companion
  Tasks class (Lesson 7).

- Domain records BookingChange, ServiceRequest, SanitizedMessage, ToolCall,
  ConfirmationRequest, ServiceOutcome, OutcomeStatus, ServiceRequestState, RequestStatus.

- ModificationGuardrail.java implementing the before-tool-call hook. On any call to
  modifyReservation or cancelReservation, reads the arguments map for confirmationToken.
  If absent or blank, returns Guardrail.reject("MISSING_CONFIRMATION_TOKEN: call
  requestConfirmation first"). If present, passes through.

- BookingService.java — in-process booking record store. Reads bookings from
  src/main/resources/sample-events/bookings.jsonl on startup. Exposes searchBooking,
  modifyReservation (requires token), cancelReservation (requires token), fileComplaint.
  Returns typed response records (BookingRecord, ModificationResult, ComplaintReceipt).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9640 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/bookings.jsonl with 5 seeded booking records covering
  seat upgrades, rebookable itineraries, and a complaint-eligible missed-connection scenario.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.

- eval-matrix.yaml at the project root with 3 controls (S1, G1, H1) matching Section 8.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  decisions.authority_level = automated-with-confirmation, oversight.human_in_loop = true.

- prompts/customer-service-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: AirlineCS", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission form + live list of request cards; right = selected-request detail with
  sanitized message preview, tool call log, confirmation panel when AWAITING_CONFIRMATION,
  and outcome summary).
  Browser title exactly: <title>Akka Sample: AirlineCS</title>. No subtitle on the
  Overview tab.

Generation workflow — Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key. MockModelProvider returns random-but-shape-correct
        ServiceOutcome objects per task iteration. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file.

Mock LLM provider — when option (a) is selected:

- Generate MockModelProvider.java per-task dispatch on HANDLE_SERVICE_REQUEST.
  Reads src/main/resources/mock-responses/handle-service-request.json. 8 entries covering
  the four OutcomeStatus values. One entry per ReAct pattern (search-only, search+modify
  with HITL, search+cancel, complaint-file). Plus 2 malformed entries: one with a missing
  confirmationToken on modifyReservation (triggering the guardrail), one with an unknown
  toolName. The mock selects a malformed entry on the first iteration of every 4th request
  (modulo seed) so J2 is reproducible.
- MockModelProvider.seedFor(requestId) makes selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: CustomerServiceAgent extends AutonomousAgent. ServiceTasks.java MUST exist.
- Lesson 4: awaitSanitizedStep 15s, serveStep 120s, hitlStep 3600s, completeStep 5s.
- Lesson 6: every Optional<T> lifecycle field on ServiceRequestState.
- Lesson 7: ServiceTasks.java with HANDLE_SERVICE_REQUEST is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9640 in application.conf akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column, no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: Mermaid CSS overrides + themeVariables in generated index.html.
- Lesson 25: never write key value to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute, exactly five tab-panel sections.
- Single-agent invariant: exactly ONE AutonomousAgent (CustomerServiceAgent). BookingService
  is a plain Java class, not an agent. ModificationGuardrail is not an agent.
- The HITL gate is application-level: the workflow pauses in hitlStep; the customer
  submits confirmation via the REST API. No external review queue or approval service.
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
