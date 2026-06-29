# SPEC — airline-triage-router

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Custom Orchestration Airline Assistant.
**One-line pitch:** A custom orchestrator classifies an inbound passenger request by intent, hands the resolution off to the matching specialist agent that owns it end-to-end, and applies three layered governance controls — PNR/PII sanitization, a before-tool-call guardrail on destructive operations, and a before-agent-response guardrail on every customer-facing draft.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *which specialist* should own the task, then transfers the same task identity to that downstream specialist, which produces the final passenger-facing response. The downstream specialist is responsible for the whole resolution; the classifier does not narrate or summarise. Three governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw request event and the LLM call. PNR locators, passport/ID numbers, frequent-flyer numbers, and email addresses are stripped before any agent sees the payload.
- A **before-tool-call guardrail** fires before any destructive tool action (seat mutation, booking cancellation, refund initiation). It checks the proposed action against a fare-rule policy and blocks it when the action exceeds the agent's authority.
- A **before-agent-response guardrail** runs on every specialist's draft response. It blocks drafts that echo a PNR token, promise a compensation amount not authorised by policy, or offer legal or medical advice outside the airline's scope.

The pattern is a fan-out-of-one: the workflow branches on the classifier's intent, and only the chosen specialist is invoked for that request.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live request list. Every card displays its intent chip, status pill, routing score, and (if resolved) the published response.
2. `RequestFeeder` (TimedAction) ticks every 30 s and inserts a new canned passenger request from `sample-events/passenger-requests.jsonl` into `PassengerRequestQueue`.
3. For each new request: `PnrSanitizer` (Consumer) redacts the payload, registers a `FlightRequestEntity`, and starts a `FlightRequestWorkflow`.
4. The workflow calls `IntentRouter`, gets an `IntentDecision { intent, confidence, reason }`, and emits `IntentClassified` on the entity.
5. Branch on `intent`:
   - `BOOKING` → workflow calls `BookingSpecialist` with the `RESOLVE` task and awaits a typed `AirlineResolution`.
   - `CHANGE` → workflow calls `ChangeSpecialist` with `RESOLVE`.
   - `BAGGAGE` → workflow calls `BaggageSpecialist` with `RESOLVE`.
   - `STATUS` → workflow calls `StatusSpecialist` with `RESOLVE`.
   - `UNCLEAR` → workflow emits `RequestUnresolved`; ends.
6. Before the specialist executes a destructive tool action (cancel, rebook, issue refund), `ToolCallGuardrail` checks the action against the fare-rule policy. If denied, the specialist halts, emits `ToolCallBlocked`, and the request lands in `BLOCKED`.
7. The specialist's draft `AirlineResolution` passes through `ResponseGuardrail`. If accepted, `ResponsePublished` is emitted (terminal `RESOLVED`). If rejected, `ResponseBlocked` is emitted (terminal `BLOCKED`) with the violation list.
8. Independent of the workflow, `RoutingEvalScorer` (Consumer) listens for `IntentClassified` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the entity.
9. The user can click any request card and see the redacted payload, the classification reason, the routing score, the chosen specialist, the draft (or blocked draft + violations), and the published response.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RequestFeeder` | `TimedAction` | Drips simulated passenger requests every 30 s. | scheduler | `PassengerRequestQueue` |
| `PassengerRequestQueue` | `EventSourcedEntity` | Append-only audit log of every inbound request (`InboundPassengerRequestReceived`). | `RequestFeeder`, `AirlineEndpoint` | `PnrSanitizer` |
| `PnrSanitizer` | `Consumer` | Subscribes to `PassengerRequestQueue` events; redacts PNR/PII; registers `FlightRequestEntity`; starts `FlightRequestWorkflow`. | `PassengerRequestQueue` events | `FlightRequestEntity`, `FlightRequestWorkflow` |
| `IntentRouter` | `Agent` (typed, not autonomous) | Classifies a `SanitizedPassengerRequest` into `BOOKING` / `CHANGE` / `BAGGAGE` / `STATUS` / `UNCLEAR` with confidence + reason. | invoked by `FlightRequestWorkflow` | returns `IntentDecision` |
| `BookingSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for booking requests. | invoked by `FlightRequestWorkflow` | returns `AirlineResolution` |
| `ChangeSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for change and cancellation requests. | invoked by `FlightRequestWorkflow` | returns `AirlineResolution` |
| `BaggageSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for baggage and lost-luggage requests. | invoked by `FlightRequestWorkflow` | returns `AirlineResolution` |
| `StatusSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for flight-status and itinerary inquiries. | invoked by `FlightRequestWorkflow` | returns `AirlineResolution` |
| `RoutingJudge` | `Agent` (typed) | Grades a classification decision against the sanitized payload. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RoutingEvalScorer` | returns `RoutingScore` |
| `ToolCallGuardrail` | `Agent` (typed) | Before-tool-call guardrail: checks a proposed destructive action against fare-rule policy. Returns `ToolCallVerdict { allowed, denialReason }`. | invoked by specialists via `FlightRequestWorkflow` | returns `ToolCallVerdict` |
| `ResponseGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks a draft `AirlineResolution` against the response-policy rubric. Returns `ResponseVerdict { allowed, violations, rubricVersion }`. | invoked by `FlightRequestWorkflow` | returns `ResponseVerdict` |
| `FlightRequestWorkflow` | `Workflow` | Per-request orchestration: classify → route → resolve → (tool-call guardrail inline) → response guardrail → publish. | `PnrSanitizer` (start) | `FlightRequestEntity` |
| `FlightRequestEntity` | `EventSourcedEntity` | Per-request lifecycle. | `FlightRequestWorkflow`, `RoutingEvalScorer` | `FlightRequestView` |
| `FlightRequestView` | `View` | Read-model row per request. | `FlightRequestEntity` events | `AirlineEndpoint` |
| `RoutingEvalScorer` | `Consumer` | Subscribes to `FlightRequestEntity` events; on `IntentClassified` invokes `RoutingJudge` and writes `RoutingScored` back. | `FlightRequestEntity` events | `FlightRequestEntity` |
| `AirlineEndpoint` | `HttpEndpoint` | `/api/requests/*` — list, get, manual submit, unblock, SSE; `/api/metadata/*`. | — | `FlightRequestView`, `FlightRequestEntity`, `PassengerRequestQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record PassengerRequest(
    String requestId,
    String passengerId,
    String channel,          // "web" | "mobile" | "phone-ivr" | "kiosk"
    String subject,
    String body,
    Instant receivedAt
) {}

record SanitizedPassengerRequest(
    String redactedSubject,
    String redactedBody,
    List<String> piiCategoriesFound
) {}

enum PassengerIntent { BOOKING, CHANGE, BAGGAGE, STATUS, UNCLEAR }

record IntentDecision(
    PassengerIntent intent,
    String confidence,       // "high" | "medium" | "low"
    String reason            // one short sentence
) {}

enum ResolutionOutcome {
    BOOKING_CONFIRMED,
    CHANGE_APPLIED,
    BAGGAGE_CLAIM_FILED,
    STATUS_PROVIDED,
    FOLLOW_UP_SCHEDULED,
    ESCALATED
}

record AirlineResolution(
    String responseSubject,
    String responseBody,
    ResolutionOutcome outcome,
    String specialistTag,    // "booking" | "change" | "baggage" | "status"
    Instant resolvedAt
) {}

record ToolCallVerdict(
    boolean allowed,
    String denialReason      // null when allowed
) {}

record ResponseVerdict(
    boolean allowed,
    List<String> violations, // empty when allowed
    String rubricVersion
) {}

record RoutingScore(
    int score,               // 1..5
    String rationale,
    Instant scoredAt
) {}

record FlightRequest(
    String requestId,
    PassengerRequest incoming,
    Optional<SanitizedPassengerRequest> sanitized,
    Optional<IntentDecision> intent,
    Optional<AirlineResolution> resolution,
    Optional<ToolCallVerdict> toolCallVerdict,
    Optional<ResponseVerdict> responseVerdict,
    Optional<RoutingScore> routingScore,
    Optional<String> unresolvedReason,
    FlightRequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum FlightRequestStatus {
    RECEIVED,
    SANITIZED,
    CLASSIFIED,
    ROUTED_BOOKING,
    ROUTED_CHANGE,
    ROUTED_BAGGAGE,
    ROUTED_STATUS,
    RESOLUTION_DRAFTED,
    BLOCKED,
    RESOLVED,
    UNRESOLVED
}
```

Events on `FlightRequestEntity`: `RequestRegistered`, `RequestSanitized`, `IntentClassified`, `RequestRouted`, `ResolutionDrafted`, `ToolCallVerdictAttached`, `ResponseVerdictAttached`, `ResponsePublished`, `ResponseBlocked`, `RequestUnresolved`, `RoutingScored`.

Events on `PassengerRequestQueue`: `InboundPassengerRequestReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/requests` — list all requests (newest-first), optional `?intent=BOOKING|CHANGE|BAGGAGE|STATUS|UNCLEAR&status=…` filtered client-side.
- `GET /api/requests/{id}` — one request.
- `POST /api/requests` — manually submit a request (body `PassengerRequest` minus `requestId` and `receivedAt`); server assigns both.
- `POST /api/requests/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `BLOCKED` to `RESOLVED` if the operator chooses to publish the blocked draft.
- `GET /api/requests/sse` — Server-Sent Events for every request state change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Custom Orchestration Airline Assistant</title>`.

The App UI tab is a three-pane layout: **left** is the request list (status pill + intent chip + routing score chip), **centre** is the selected request's redacted payload + classification decision + score, **right** is the chosen specialist's draft + tool-call verdict + response guardrail verdict + published response (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PnrSanitizer` Consumer): strips PNR locators, passport and national-ID numbers, frequent-flyer membership numbers, and email addresses from the payload before any LLM sees it. The categories found are kept for audit; only the redacted text reaches agents.
- **G1 — before-tool-call guardrail** on all four specialists: checks every proposed destructive action (cancel booking, rebook onto a different fare, issue refund) against a fare-rule policy matrix before the tool executes. Blocking — a denial puts the request in `BLOCKED` for operator review.
- **G2 — before-agent-response guardrail** on all four specialists: checks every draft `AirlineResolution` against a response rubric (no echoed PNR tokens, no compensation amounts beyond policy limits, no medical or legal advice). Blocking — a violation puts the request in `BLOCKED`.
- **E1 — on-decision eval** (`eval-event`, on the intent classification): `RoutingEvalScorer` (Consumer) listens for `IntentClassified` events and calls `RoutingJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking.

## 9. Agent prompts

- `IntentRouter` → `prompts/intent-router.md`. Typed classifier; returns one of `BOOKING`, `CHANGE`, `BAGGAGE`, `STATUS`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `BookingSpecialist` → `prompts/booking-specialist.md`. Owns the `RESOLVE` task for new and rebook requests.
- `ChangeSpecialist` → `prompts/change-specialist.md`. Owns the `RESOLVE` task for flight changes and cancellations.
- `BaggageSpecialist` → `prompts/baggage-specialist.md`. Owns the `RESOLVE` task for baggage-claim and lost-luggage disputes.
- `StatusSpecialist` → `prompts/status-specialist.md`. Owns the `RESOLVE` task for flight and itinerary status inquiries.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a classification decision against a 1–5 rubric with one-sentence rationale.
- `ToolCallGuardrail` → `prompts/tool-call-guardrail.md`. Returns `ToolCallVerdict { allowed, denialReason }`. Conservative — fare-rule edge cases are denied.
- `ResponseGuardrail` → `prompts/response-guardrail.md`. Returns `ResponseVerdict { allowed, violations }`. Conservative — borderline drafts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Feeder drips a booking-intent request → classified `BOOKING` → resolved by `BookingSpecialist` → response guardrail passes → published.
2. **J2** — Feeder drips a baggage-claim request → classified `BAGGAGE` → resolved by `BaggageSpecialist` → published.
3. **J3** — An ambiguous request classifies as `UNCLEAR` and lands in `UNRESOLVED` without any specialist invocation.
4. **J4** — A change request on a non-refundable fare triggers the before-tool-call guardrail; the request lands in `BLOCKED` with the denial reason.
5. **J5** — A draft that echoes a PNR locator token is blocked by the response guardrail; the operator can unblock or leave it.
6. **J6** — Every classified request carries a `RoutingScore` (1–5) and rationale within ~10 s of the classification event.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named airline-triage-router demonstrating the handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real airline PSS integration).
Maven group io.akka.samples. Maven artifact handoff-routing-cx-support-airline-triage-router.
Java package io.akka.samples.customorchestrationairlineassistant. Akka 3.6.0. HTTP port 9231.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) IntentRouter — classifier. System prompt loaded from
  prompts/intent-router.md. Input: SanitizedPassengerRequest{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}. Output: IntentDecision{intent: PassengerIntent
  (BOOKING/CHANGE/BAGGAGE/STATUS/UNCLEAR), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent BookingSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(4)). System prompt from prompts/booking-specialist.md. Input:
  SanitizedPassengerRequest + IntentDecision. Output: AirlineResolution{responseSubject,
  responseBody, outcome: ResolutionOutcome, specialistTag = "booking", resolvedAt}.

- 1 AutonomousAgent ChangeSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(4)). System prompt from prompts/change-specialist.md. Same input shape;
  specialistTag = "change".

- 1 AutonomousAgent BaggageSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(4)). System prompt from prompts/baggage-specialist.md. Same input;
  specialistTag = "baggage".

- 1 AutonomousAgent StatusSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(2)). System prompt from prompts/status-specialist.md. Same input;
  specialistTag = "status".

- 1 Agent (typed) RoutingJudge — judge. System prompt from prompts/routing-judge.md. Input:
  SanitizedPassengerRequest + IntentDecision. Output: RoutingScore{score: int 1–5,
  rationale: String, scoredAt: Instant}.

- 1 Agent (typed) ToolCallGuardrail — typed fare-rule check. System prompt from
  prompts/tool-call-guardrail.md. Input: SanitizedPassengerRequest + proposed action String.
  Output: ToolCallVerdict{allowed: boolean, denialReason: String (null when allowed)}.
  Used by FlightRequestWorkflow before any destructive specialist tool call; blocking.

- 1 Agent (typed) ResponseGuardrail — typed response check. System prompt from
  prompts/response-guardrail.md. Input: SanitizedPassengerRequest + AirlineResolution.
  Output: ResponseVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by FlightRequestWorkflow before publishing; blocking.

- 1 Workflow FlightRequestWorkflow per requestId. Steps:
    classifyStep -> routeStep -> {bookingStep|changeStep|baggageStep|statusStep|unresolvedStep}
                -> responseGuardrailStep -> publishStep
  classifyStep calls componentClient.forAgent().inSession(requestId)
    .method(IntentRouter::classify).invoke(sanitized). On success emits IntentClassified
    via FlightRequestEntity.recordIntent.
  routeStep branches on IntentDecision.intent:
    BOOKING   -> proceed to bookingStep  (emits RequestRouted{BOOKING})
    CHANGE    -> proceed to changeStep   (emits RequestRouted{CHANGE})
    BAGGAGE   -> proceed to baggageStep  (emits RequestRouted{BAGGAGE})
    STATUS    -> proceed to statusStep   (emits RequestRouted{STATUS})
    UNCLEAR   -> unresolvedStep (emits RequestUnresolved; terminates).
  bookingStep / changeStep / baggageStep / statusStep call
    forAutonomousAgent(<Specialist>.class, requestId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, intent))) returning a taskId,
    then forTask(taskId).result(AirlineTasks.RESOLVE) to block on the typed AirlineResolution.
    Specialists may call ToolCallGuardrail internally before any destructive tool action;
    on ToolCallVerdict.allowed=false, the specialist emits ToolCallVerdictAttached (denied)
    and returns a Resolution with outcome=ESCALATED. Workflow emits ResolutionDrafted.
  responseGuardrailStep calls forAgent(...).method(ResponseGuardrail::check)
    .invoke(sanitized, draft). On verdict.allowed=true proceed to publishStep (emits
    ResponsePublished, terminal RESOLVED). On verdict.allowed=false emit ResponseBlocked
    (terminal BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on classifyStep and
    responseGuardrailStep, stepTimeout(Duration.ofSeconds(60)) on bookingStep, changeStep,
    baggageStep, statusStep, publishStep. defaultStepRecovery(maxRetries(2).failoverTo(
    FlightRequestWorkflow::error)).

- 2 EventSourcedEntities:
    * PassengerRequestQueue — append-only audit log. Command receive(PassengerRequest) emits
      InboundPassengerRequestReceived{incoming}. No mutable state beyond a counter; commands
      are idempotent on incoming.requestId.
    * FlightRequestEntity (one per requestId) — full per-request lifecycle. State
      FlightRequest{requestId, incoming: PassengerRequest,
      Optional<SanitizedPassengerRequest> sanitized,
      Optional<IntentDecision> intent, Optional<AirlineResolution> resolution,
      Optional<ToolCallVerdict> toolCallVerdict, Optional<ResponseVerdict> responseVerdict,
      Optional<RoutingScore> routingScore, Optional<String> unresolvedReason,
      FlightRequestStatus status, Instant createdAt, Optional<Instant> finishedAt}.
      FlightRequestStatus enum: RECEIVED, SANITIZED, CLASSIFIED, ROUTED_BOOKING, ROUTED_CHANGE,
      ROUTED_BAGGAGE, ROUTED_STATUS, RESOLUTION_DRAFTED, BLOCKED, RESOLVED, UNRESOLVED.
      Events: RequestRegistered, RequestSanitized, IntentClassified, RequestRouted,
      ResolutionDrafted, ToolCallVerdictAttached, ResponseVerdictAttached, ResponsePublished,
      ResponseBlocked, RequestUnresolved, RoutingScored. Commands: registerIncoming,
      attachSanitized, recordIntent, recordRouting, recordDraft, attachToolCallVerdict,
      attachResponseVerdict, publish, block, markUnresolved, recordRoutingScore, unblock,
      getRequest. emptyState() returns FlightRequest.initial("") with no commandContext()
      reference.

- 2 Consumers:
    * PnrSanitizer subscribed to PassengerRequestQueue events; for each
      InboundPassengerRequestReceived applies a regex+heuristic redaction pipeline (6-char
      alphanumeric PNR locators matching [A-Z]{6}\b, passport numbers matching [A-Z]{1,2}\d{6,9},
      frequent-flyer numbers matching FF-\w+, email addresses, phone numbers) to subject + body,
      builds SanitizedPassengerRequest with piiCategoriesFound, and calls FlightRequestEntity
      .registerIncoming then attachSanitized for the requestId; then starts a FlightRequestWorkflow
      with requestId as the workflow id.
    * RoutingEvalScorer subscribed to FlightRequestEntity events; on IntentClassified invokes
      RoutingJudge.score(sanitized, decision) and calls FlightRequestEntity.recordRoutingScore(
      requestId, score). On any other event type, no-op. Use componentClient.

- 1 View FlightRequestView with row type FlightRequestRow (mirrors FlightRequest; uses
  Optional<T> for every nullable lifecycle field per Lesson 6). Table updater consumes
  FlightRequestEntity events. ONE query getAllRequests SELECT * AS requests FROM
  flight_request_view. No WHERE intent or WHERE status filter — filter client-side.

- 1 TimedAction RequestFeeder — every 30 s, reads next line from
  src/main/resources/sample-events/passenger-requests.jsonl (loops at EOF) and calls
  PassengerRequestQueue.receive with a fresh requestId (UUID).

- 2 HttpEndpoints:
    * AirlineEndpoint at /api with GET /requests (list from FlightRequestView.getAllRequests,
      filter client-side by ?intent and ?status query params), GET /requests/{id},
      POST /requests (body PassengerRequest minus requestId/receivedAt — server assigns),
      POST /requests/{id}/unblock (body {decidedBy, note} — operator override:
      publishes the blocked draft as RESOLVED with an audit note),
      GET /requests/sse (serverSentEventsForView over getAllRequests), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- AirlineTasks.java declaring the task constant: RESOLVE (resultConformsTo AirlineResolution.class,
  description "Resolve the passenger request end-to-end and return a typed AirlineResolution").
- Domain records PassengerRequest, SanitizedPassengerRequest, IntentDecision, AirlineResolution,
  ToolCallVerdict, ResponseVerdict, RoutingScore, and the FlightRequest entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9231 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/passenger-requests.jsonl with 10 canned lines (3 BOOKING,
  2 CHANGE, 2 BAGGAGE, 2 STATUS, 1 UNCLEAR).
  One CHANGE line is engineered to trigger the before-tool-call guardrail (non-refundable
  fare cancellation request). One BOOKING line includes a PNR locator in the draft path to
  trip the response guardrail.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 4 controls: S1 sanitizer pii, G1 guardrail
  before-tool-call fare-rule, G2 guardrail before-agent-response response-rubric, E1
  eval-event on-decision-eval. Matching simplified_view list.
- risk-survey.yaml at the project root with purpose.primary_function = customer-support,
  data.data_classes.pii = true, decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false, data.pii_handled_by_sanitizer_before_llm = true,
  failure.failure_modes including "wrong-intent-routing", "pnr-leakage-via-llm",
  "non-refundable-fare-cancellation", "compensation-fabrication"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- Prompt files: prompts/intent-router.md, prompts/booking-specialist.md,
  prompts/change-specialist.md, prompts/baggage-specialist.md, prompts/status-specialist.md,
  prompts/routing-judge.md, prompts/tool-call-guardrail.md, prompts/response-guardrail.md.
- README.md at the project root: title "Akka Sample: Custom Orchestration Airline Assistant",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file. Five tabs
  matching the formal exemplar. App UI tab uses a three-column layout (left = request list
  with intent chip + status pill + score chip; centre = redacted request + classification
  block; right = specialist draft + tool-call verdict + response guardrail verdict + published
  response or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: Custom Orchestration Airline Assistant</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — generate a MockModelProvider with per-agent dispatch.
        Per-agent mock-response shapes:
        intent-router.json — 12 IntentDecision entries spanning BOOKING (seat selection,
          fare upgrade, group booking), CHANGE (rebooking, cancellation, name correction),
          BAGGAGE (delayed bag claim, excess baggage fee, lost item), STATUS (departure
          status, gate change), and UNCLEAR (vague message, wrong-language input). Each
          with confidence and one-sentence reason.
        booking-specialist.json — 8 AirlineResolution entries: 5 BOOKING_CONFIRMED,
          1 FOLLOW_UP_SCHEDULED, 1 ESCALATED, 1 designed to echo a PNR locator token
          to trip the response guardrail.
        change-specialist.json — 8 AirlineResolution entries: 4 CHANGE_APPLIED,
          2 ESCALATED, 1 FOLLOW_UP_SCHEDULED, 1 on a non-refundable fare designed to
          produce a ToolCallVerdict denied result.
        baggage-specialist.json — 8 AirlineResolution entries: 4 BAGGAGE_CLAIM_FILED,
          2 STATUS_PROVIDED (baggage status update), 1 ESCALATED, 1 with an invented
          compensation amount to trip the response guardrail.
        status-specialist.json — 6 AirlineResolution entries: 4 STATUS_PROVIDED,
          1 FOLLOW_UP_SCHEDULED, 1 ESCALATED.
        routing-judge.json — 10 RoutingScore entries, score 1–5, one-sentence rationale.
        tool-call-guardrail.json — 6 ToolCallVerdict entries: 4 allowed=true,
          2 allowed=false with denialReason ("non-refundable-fare", "outside-change-window").
        response-guardrail.json — 10 ResponseVerdict entries: 7 allowed=true, 3
          allowed=false: "echoes-pnr-token", "invented-compensation-amount",
          "out-of-scope-legal-advice".
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent. All four specialists
  extend akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): classifyStep 20s,
  responseGuardrailStep 20s, booking/change/baggage/status/publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on FlightRequest is Optional<T>.
- (Lesson 7) AirlineTasks.java declares the RESOLVE Task<AirlineResolution> constant.
  All four specialists' definition().capability(TaskAcceptance.of(RESOLVE)...) reference it.
- (Lesson 8) Model names: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere.
- (Lesson 10) Port 9231 in application.conf.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier label is "Runs out of the box".
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides AND theme
  variables (state-diagram label colour white, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. No hidden zombie panels.
- The PnrSanitizer runs INSIDE a Consumer before any LLM call.
- The RoutingEvalScorer Consumer reacts to IntentClassified events and calls RoutingJudge
  via componentClient.forAgent(). It does NOT modify the workflow flow.
- The before-tool-call guardrail step happens BEFORE any destructive tool action fires.
- The response guardrail step happens BEFORE ResponsePublished. A blocked draft never
  reaches the UI as published.
- No forbidden words in user-facing text.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
