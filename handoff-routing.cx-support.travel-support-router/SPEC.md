# SPEC — travel-support-router

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Travel Support Router.
**One-line pitch:** A router agent classifies an incoming travel-support request and hands the resolution off to a flights, hotels, car rental, or excursions specialist that owns it end-to-end, with PII redaction before any LLM call, a before-tool-call guardrail on booking mutations, and an application-level HITL gate that pauses sensitive changes for passenger confirmation.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *which* specialist should own the task, then transfers the same task identity to that specialist who produces the final output. The classifier does not narrate or summarise. Three governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw request event and the LLM call. The classifier and all specialists never see raw passenger names, passport numbers, loyalty account numbers, or payment-card digits.
- A **before-tool-call guardrail** fires every time a specialist agent prepares a booking or cancellation tool invocation. It checks the proposed action against a travel-policy rubric (no non-refundable cancellations outside the 24-hour window, no fare-class upgrades above the passenger's entitlement, no waiver of mandatory travel-insurance prompts) and blocks the tool call when it violates, putting the request in `BLOCKED` for operator review.
- An **application-level HITL** gate pauses the workflow after a specialist produces a proposed booking change and before any mutation executes. A plain-language confirmation summary is presented to the passenger. On `CONFIRM` the workflow resumes and commits the change; on `REJECT` the workflow terminates in `PASSENGER_REJECTED` without any mutation.

The pattern is a fan-out-of-one: the workflow branches on the classifier's category, and only the chosen specialist handles that request. The other specialists see no traffic for it.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live request list. Every request displays its category chip, status pill, routing score, and (if resolved) the published response.
2. `RequestSimulator` (TimedAction) ticks every 30 s and inserts a new canned request from `sample-events/travel-requests.jsonl` into `RequestQueue`.
3. For each new request: `PiiSanitizer` (Consumer) redacts the payload, registers a `RequestEntity`, and starts a `TravelWorkflow`.
4. The workflow calls `RouterAgent`, gets a `RoutingDecision { category, confidence, reason }`, and emits `RoutingDecided` on the entity.
5. Branch on `category`:
   - `FLIGHTS` → workflow calls `FlightSpecialist` with the `RESOLVE` task.
   - `HOTELS` → workflow calls `HotelSpecialist`.
   - `CAR_RENTAL` → workflow calls `CarRentalSpecialist`.
   - `EXCURSIONS` → workflow calls `ExcursionSpecialist`.
   - `UNCLEAR` → workflow emits `RequestEscalated`; ends in `ESCALATED`.
6. Before the specialist executes any booking/cancellation tool, `BookingGuardrail` checks the proposed tool call. If blocked → `BLOCKED`. If allowed → continue.
7. A `ConfirmationGate` step presents a plain-language change summary to the passenger. On `CONFIRM` the specialist's `Resolution` passes through to publish (`RESOLVED`). On `REJECT` the workflow emits `PassengerRejected` (terminal `PASSENGER_REJECTED`).
8. Independent of the workflow, `RouterEvalScorer` (Consumer) listens for `RoutingDecided` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the entity.
9. The user can click any request card and see the redacted request, the routing reason, the routing score, the chosen specialist, the proposed change summary, the guardrail verdict, and the published response.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RequestSimulator` | `TimedAction` | Drips simulated travel-support requests into `RequestQueue` every 30 s. | scheduler | `RequestQueue` |
| `RequestQueue` | `EventSourcedEntity` | Append-only audit log of every inbound request (`InboundRequestReceived`). | `RequestSimulator`, `TravelEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Subscribes to `RequestQueue` events; redacts PII; registers `RequestEntity`; starts a `TravelWorkflow`. | `RequestQueue` events | `RequestEntity`, `TravelWorkflow` |
| `RouterAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedRequest` into `FLIGHTS` / `HOTELS` / `CAR_RENTAL` / `EXCURSIONS` / `UNCLEAR` with confidence + reason. | invoked by `TravelWorkflow` | returns `RoutingDecision` |
| `FlightSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for flight-related requests. Returns typed `Resolution`. | invoked by `TravelWorkflow` | returns `Resolution` |
| `HotelSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for hotel-related requests. Returns typed `Resolution`. | invoked by `TravelWorkflow` | returns `Resolution` |
| `CarRentalSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for car-rental requests. Returns typed `Resolution`. | invoked by `TravelWorkflow` | returns `Resolution` |
| `ExcursionSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for excursion and activity requests. Returns typed `Resolution`. | invoked by `TravelWorkflow` | returns `Resolution` |
| `RoutingJudge` | `Agent` (typed) | Grades a routing decision against the sanitized request. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RouterEvalScorer` | returns `RoutingScore` |
| `BookingGuardrail` | `Agent` (typed) | Before-tool-call guardrail: checks a proposed booking/cancellation action against the travel-policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `TravelWorkflow` | returns `GuardrailVerdict` |
| `ConfirmationGate` | `Agent` (typed) | Application HITL: produces a plain-language change summary for the passenger. Returns `ConfirmationRequest { summary, deadline }`. Workflow pauses awaiting `CONFIRM` or `REJECT`. | invoked by `TravelWorkflow` | returns `ConfirmationRequest` |
| `TravelWorkflow` | `Workflow` | Per-request orchestration: route → specialize → guardrail → confirm → publish. | `PiiSanitizer` (start) | `RequestEntity` |
| `RequestEntity` | `EventSourcedEntity` | Per-request lifecycle. | `TravelWorkflow`, `RouterEvalScorer` | `RequestView` |
| `RequestView` | `View` | Read-model row per request. | `RequestEntity` events | `TravelEndpoint` |
| `RouterEvalScorer` | `Consumer` | Subscribes to `RequestEntity` events; on `RoutingDecided` invokes `RoutingJudge` and writes `RoutingScored` back. | `RequestEntity` events | `RequestEntity` |
| `TravelEndpoint` | `HttpEndpoint` | `/api/requests/*` — list, get, manual submit, manual unblock, passenger confirm/reject, SSE; `/api/metadata/*`. | — | `RequestView`, `RequestEntity`, `RequestQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record IncomingRequest(
    String requestId,
    String passengerId,
    String channel,           // "web-chat" | "app" | "phone-transcript"
    String subject,
    String body,
    Instant receivedAt
) {}

record SanitizedRequest(
    String redactedSubject,
    String redactedBody,
    List<String> piiCategoriesFound
) {}

enum TravelCategory { FLIGHTS, HOTELS, CAR_RENTAL, EXCURSIONS, UNCLEAR }

record RoutingDecision(
    TravelCategory category,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

enum ResolutionAction {
    BOOKING_CHANGED, BOOKING_CANCELLED, BOOKING_CONFIRMED,
    INFO_PROVIDED, ARTICLE_LINKED, FOLLOW_UP_SCHEDULED, ESCALATED
}

record Resolution(
    String responseSubject,
    String responseBody,
    ResolutionAction action,
    String specialistTag,      // "flights" | "hotels" | "car-rental" | "excursions"
    Optional<String> bookingRef,
    Instant resolvedAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,   // empty when allowed
    String rubricVersion
) {}

record ConfirmationRequest(
    String summary,            // plain-language change description for the passenger
    Instant deadline           // when the confirmation window closes
) {}

enum ConfirmationOutcome { CONFIRM, REJECT }

record RoutingScore(
    int score,                 // 1..5
    String rationale,
    Instant scoredAt
) {}

record TravelRequest(
    String requestId,
    IncomingRequest incoming,
    Optional<SanitizedRequest> sanitized,
    Optional<RoutingDecision> routing,
    Optional<GuardrailVerdict> guardrail,
    Optional<ConfirmationRequest> confirmation,
    Optional<ConfirmationOutcome> confirmationOutcome,
    Optional<Resolution> resolution,
    Optional<RoutingScore> routingScore,
    Optional<String> escalationReason,
    RequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus {
    RECEIVED,
    SANITIZED,
    ROUTED,
    ROUTED_FLIGHTS,
    ROUTED_HOTELS,
    ROUTED_CAR_RENTAL,
    ROUTED_EXCURSIONS,
    GUARDRAIL_CLEARED,
    AWAITING_CONFIRMATION,
    RESOLUTION_DRAFTED,
    BLOCKED,
    RESOLVED,
    PASSENGER_REJECTED,
    ESCALATED
}
```

Events on `RequestEntity`: `RequestRegistered`, `RequestSanitized`, `RoutingDecided`, `RequestRouted`, `GuardrailVerdictAttached`, `ConfirmationRequested`, `PassengerConfirmed`, `PassengerRejected`, `ResolutionDrafted`, `ResolutionPublished`, `ResolutionBlocked`, `RequestEscalated`, `RoutingScored`.

Events on `RequestQueue`: `InboundRequestReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/requests` — list all requests (newest-first), optional `?category=FLIGHTS|HOTELS|CAR_RENTAL|EXCURSIONS|UNCLEAR&status=…` filtered client-side.
- `GET /api/requests/{id}` — one request.
- `POST /api/requests` — manually submit a request (body `IncomingRequest` minus `requestId` and `receivedAt`); server assigns both.
- `POST /api/requests/{id}/confirm` — body `{ "outcome": "CONFIRM" | "REJECT", "passengerId": String }` — passenger response at the HITL gate.
- `POST /api/requests/{id}/unblock` — body `{ "decidedBy": String, "note": String }` — operator override; transitions `BLOCKED` to `RESOLVED` if the operator chooses to publish.
- `GET /api/requests/sse` — Server-Sent Events for every request change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Travel Support Router</title>`.

The App UI tab is a three-pane layout: **left** is the request list (status pill + category chip + routing score chip), **centre** is the selected request's redacted body + routing decision + score, **right** is the chosen specialist's proposed resolution + guardrail verdict + confirmation state + published response (or violations + Unblock button when `BLOCKED`, or passenger-rejected indicator when `PASSENGER_REJECTED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts passenger names, passport numbers, loyalty-program account numbers, and payment-card digits from the request before any LLM sees it.
- **G1 — before-tool-call guardrail** on all four specialists: checks every proposed booking or cancellation tool call against a travel-policy rubric (no non-refundable cancellations outside the 24-hour window, no fare-class upgrades above entitlement, no bypassing mandatory insurance prompts). Blocking — a violation puts the request in `BLOCKED` for operator review.
- **H1 — application HITL** on all booking-mutation specialists: after a specialist produces a proposed change and before any mutation executes, `ConfirmationGate` produces a plain-language summary, the workflow pauses, and the passenger must `CONFIRM` or `REJECT`. A `REJECT` terminates the workflow in `PASSENGER_REJECTED` with no mutation applied.

## 9. Agent prompts

- `RouterAgent` → `prompts/router-agent.md`. Typed classifier; returns one of `FLIGHTS`, `HOTELS`, `CAR_RENTAL`, `EXCURSIONS`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `FlightSpecialist` → `prompts/flight-specialist.md`. Owns the `RESOLVE` task for flight requests. Never books a seat class above the passenger's entitlement; routes fare-waiver requests to `ESCALATED`.
- `HotelSpecialist` → `prompts/hotel-specialist.md`. Owns the `RESOLVE` task for hotel requests. Never waives cancellation fees outside policy.
- `CarRentalSpecialist` → `prompts/car-rental-specialist.md`. Owns the `RESOLVE` task for car-rental requests. Never extends a rental beyond the contracted insurance period without explicit re-confirmation.
- `ExcursionSpecialist` → `prompts/excursion-specialist.md`. Owns the `RESOLVE` task for excursion requests. Never invents availability or confirms full refunds outside the stated cancellation window.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a routing decision against a 1–5 rubric with one-sentence rationale.
- `BookingGuardrail` → `prompts/booking-guardrail.md`. Returns a `GuardrailVerdict { allowed, violations }`. Conservative — borderline calls are blocked.
- `ConfirmationGate` → `prompts/confirmation-gate.md`. Produces a plain-language `ConfirmationRequest { summary, deadline }`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a flight-rebooking request → routed `FLIGHTS` → guardrail clears → passenger confirms → `FlightSpecialist` resolves → published.
2. **J2** — Hotel check-out extension → routed `HOTELS` → resolved by `HotelSpecialist`.
3. **J3** — Ambiguous request routes `UNCLEAR` → `ESCALATED`; no specialist invoked.
4. **J4** — A specialist attempts a non-refundable cancellation outside the 24-hour window → guardrail blocks → `BLOCKED`.
5. **J5** — Passenger rejects the proposed flight change at the confirmation gate → `PASSENGER_REJECTED`; no booking mutation.
6. **J6** — Every routed request carries a `RoutingScore` (1–5) within ~10 s of the routing decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named travel-support-router demonstrating the handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real GDS or CRM integration).
Maven group io.akka.samples. Maven artifact handoff-routing-cx-support-travel-support-router.
Java package io.akka.samples.customersupportbot. Akka 3.6.0. HTTP port 9236.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) RouterAgent — classifier. System prompt loaded from
  prompts/router-agent.md. Input: SanitizedRequest{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}. Output: RoutingDecision{category: TravelCategory
  (FLIGHTS/HOTELS/CAR_RENTAL/EXCURSIONS/UNCLEAR), confidence: "high"|"medium"|"low",
  reason: String}. Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent FlightSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(4)). System prompt from prompts/flight-specialist.md. Input:
  SanitizedRequest + RoutingDecision. Output: Resolution{responseSubject, responseBody,
  action: ResolutionAction, specialistTag = "flights", bookingRef: Optional<String>,
  resolvedAt}. Never books above entitlement; sets action=ESCALATED when outside authority.

- 1 AutonomousAgent HotelSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(4)). System prompt from prompts/hotel-specialist.md. Same input
  shape; specialistTag = "hotels".

- 1 AutonomousAgent CarRentalSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(4)). System prompt from prompts/car-rental-specialist.md. Same
  input shape; specialistTag = "car-rental".

- 1 AutonomousAgent ExcursionSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/excursion-specialist.md. Same
  input shape; specialistTag = "excursions".

- 1 Agent (typed) RoutingJudge — judge. System prompt from prompts/routing-judge.md. Input:
  SanitizedRequest + RoutingDecision. Output: RoutingScore{score: int 1–5, rationale:
  String, scoredAt: Instant}.

- 1 Agent (typed) BookingGuardrail — before-tool-call rubric check. System prompt from
  prompts/booking-guardrail.md. Input: SanitizedRequest + proposed action description (String).
  Output: GuardrailVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by TravelWorkflow before any booking mutation; blocking.

- 1 Agent (typed) ConfirmationGate — HITL gate. System prompt from prompts/confirmation-gate.md.
  Input: SanitizedRequest + Resolution draft. Output: ConfirmationRequest{summary: String,
  deadline: Instant}. Used by TravelWorkflow to produce a passenger-readable change summary
  before committing any mutation.

- 1 Workflow TravelWorkflow per requestId. Steps:
    routeStep -> specialistStep (branch on category) -> guardrailStep -> confirmStep
              -> publishStep | rejectStep | escalateStep
  routeStep calls componentClient.forAgent().inSession(requestId)
    .method(RouterAgent::route).invoke(sanitized). On success emits RoutingDecided via
    RequestEntity.recordRouting.
  specialistStep branches on RoutingDecision.category:
    FLIGHTS   -> forAutonomousAgent(FlightSpecialist.class, requestId).runSingleTask(...)
    HOTELS    -> forAutonomousAgent(HotelSpecialist.class, requestId).runSingleTask(...)
    CAR_RENTAL-> forAutonomousAgent(CarRentalSpecialist.class, requestId).runSingleTask(...)
    EXCURSIONS-> forAutonomousAgent(ExcursionSpecialist.class, requestId).runSingleTask(...)
    UNCLEAR   -> escalateStep (emits RequestEscalated; terminates in ESCALATED)
  guardrailStep calls forAgent(BookingGuardrail.class, requestId).method(...).invoke(
    sanitized, proposedActionDescription). On allowed=false emits ResolutionBlocked
    (terminal BLOCKED) and ends. On allowed=true emits GuardrailVerdictAttached and proceeds.
  confirmStep calls forAgent(ConfirmationGate.class, requestId).method(...).invoke(
    sanitized, draft). Emits ConfirmationRequested and pauses. Resumes on POST
    /api/requests/{id}/confirm; on REJECT emits PassengerRejected (terminal
    PASSENGER_REJECTED) and ends; on CONFIRM emits PassengerConfirmed and proceeds.
  publishStep emits ResolutionDrafted then ResolutionPublished (terminal RESOLVED).
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on routeStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on specialistStep, confirmStep,
    and publishStep. defaultStepRecovery(maxRetries(2).failoverTo(TravelWorkflow::error)).

- 2 EventSourcedEntities:
    * RequestQueue — append-only audit log. Command receive(IncomingRequest) emits
      InboundRequestReceived{incoming}. No mutable state beyond a counter; idempotent on
      incoming.requestId.
    * RequestEntity (one per requestId) — full per-request lifecycle. State
      TravelRequest{requestId, incoming, Optional<SanitizedRequest> sanitized,
      Optional<RoutingDecision> routing, Optional<GuardrailVerdict> guardrail,
      Optional<ConfirmationRequest> confirmation, Optional<ConfirmationOutcome>
      confirmationOutcome, Optional<Resolution> resolution,
      Optional<RoutingScore> routingScore, Optional<String> escalationReason,
      RequestStatus status, Instant createdAt, Optional<Instant> finishedAt}.
      RequestStatus enum: RECEIVED, SANITIZED, ROUTED, ROUTED_FLIGHTS, ROUTED_HOTELS,
      ROUTED_CAR_RENTAL, ROUTED_EXCURSIONS, GUARDRAIL_CLEARED, AWAITING_CONFIRMATION,
      RESOLUTION_DRAFTED, BLOCKED, RESOLVED, PASSENGER_REJECTED, ESCALATED.
      Events: RequestRegistered, RequestSanitized, RoutingDecided, RequestRouted,
      GuardrailVerdictAttached, ConfirmationRequested, PassengerConfirmed,
      PassengerRejected, ResolutionDrafted, ResolutionPublished, ResolutionBlocked,
      RequestEscalated, RoutingScored.
      Commands: registerIncoming, attachSanitized, recordRouting, recordRoutedCategory,
      recordGuardrailVerdict, recordConfirmationRequest, recordConfirmationOutcome,
      recordDraft, publish, block, escalate, recordRoutingScore, unblock, getRequest.
      emptyState() returns TravelRequest.initial("") with no commandContext() reference.

- 2 Consumers:
    * PiiSanitizer subscribed to RequestQueue events; for each InboundRequestReceived
      applies a regex+heuristic redaction pipeline (passenger names in greeting positions,
      passport numbers matching [A-Z]{1,2}\d{6,9}, loyalty-program account numbers
      matching [A-Z]{2,3}\d{6,12}, payment-card numbers via Luhn-aware regex,
      booking references matching [A-Z]{2}\d{6}) to subject + body, builds
      SanitizedRequest with piiCategoriesFound, and calls RequestEntity.registerIncoming
      then attachSanitized for the requestId; then starts a TravelWorkflow with requestId
      as the workflow id.
    * RouterEvalScorer subscribed to RequestEntity events; on RoutingDecided invokes
      RoutingJudge.score(sanitized, decision) and calls RequestEntity.recordRoutingScore(
      requestId, score). On any other event type, no-op. Use componentClient.

- 1 View RequestView with row type TravelRequest (mirrors TravelRequest; uses Optional<T>
  for every nullable lifecycle field per Lesson 6). Table updater consumes RequestEntity
  events. ONE query getAllRequests SELECT * AS requests FROM request_view. No WHERE
  category or WHERE status filter — filter client-side in callers.

- 1 TimedAction RequestSimulator — every 30s, reads next line from
  src/main/resources/sample-events/travel-requests.jsonl (loops at EOF) and calls
  RequestQueue.receive with a fresh requestId (UUID).

- 2 HttpEndpoints:
    * TravelEndpoint at /api with GET /requests (list from RequestView.getAllRequests,
      filter client-side by ?category and ?status query params), GET /requests/{id},
      POST /requests (body IncomingRequest minus requestId/receivedAt — server assigns),
      POST /requests/{id}/confirm (body {outcome: "CONFIRM"|"REJECT", passengerId: String}
      — resume the paused TravelWorkflow with the passenger's choice),
      POST /requests/{id}/unblock (body {decidedBy, note} — operator override:
      publishes the blocked draft as RESOLVED with an audit note),
      GET /requests/sse (serverSentEventsForView over getAllRequests), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- TravelTasks.java declaring the task constants: RESOLVE (resultConformsTo Resolution.class,
  description "Resolve the travel-support request end-to-end and return a typed Resolution").
- Domain records IncomingRequest, SanitizedRequest, RoutingDecision, Resolution,
  GuardrailVerdict, ConfirmationRequest, RoutingScore, and the TravelRequest entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9236 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/travel-requests.jsonl with 10 canned lines (3 flight-
  flavoured, 2 hotel-flavoured, 2 car-rental-flavoured, 1 excursion-flavoured,
  1 UNCLEAR/ambiguous, 1 designed to trip the guardrail with a non-refundable
  cancellation outside the 24-hour policy window).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls: S1 sanitizer pii, G1 guardrail
  before-tool-call travel-policy-rubric, H1 hitl application passenger-confirmation.
  Matching simplified_view list. No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = customer-support,
  data.data_classes.pii = true, decisions.authority_level = hitl-confirmed,
  oversight.human_in_loop = true (passengers confirm every booking mutation),
  data.pii_handled_by_sanitizer_before_llm = true,
  failure.failure_modes including "wrong-category-routing", "unauthorized-booking-mutation",
  "pii-leakage-via-llm", "policy-violation-in-tool-call", "confirmation-bypass";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/router-agent.md, prompts/flight-specialist.md, prompts/hotel-specialist.md,
  prompts/car-rental-specialist.md, prompts/excursion-specialist.md,
  prompts/routing-judge.md, prompts/booking-guardrail.md, prompts/confirmation-gate.md
  loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Travel Support Router", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = request list with category chip + status pill + routing score chip; centre =
  redacted request + routing decision block; right = specialist draft + guardrail verdict +
  confirmation status + published response or violations + Unblock button, or
  PASSENGER_REJECTED indicator). Browser title exactly:
  <title>Akka Sample: Travel Support Router</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
        triage-agent.json → router-agent.json: 14 RoutingDecision entries spanning
          FLIGHTS (seat upgrade, rebooking, cancellation), HOTELS (early check-in,
          late check-out, room-type change), CAR_RENTAL (extension, upgrade class),
          EXCURSIONS (availability check, cancellation), UNCLEAR (mixed, very short,
          off-topic). Confidence + one-sentence reason on each.
        flight-specialist.json: 8 Resolution entries spanning BOOKING_CHANGED,
          BOOKING_CONFIRMED, INFO_PROVIDED, ESCALATED, and 1 designed to trip
          the guardrail (non-refundable cancellation outside 24h window).
        hotel-specialist.json: 6 Resolution entries: INFO_PROVIDED, BOOKING_CHANGED,
          FOLLOW_UP_SCHEDULED, ESCALATED.
        car-rental-specialist.json: 5 Resolution entries: BOOKING_CHANGED,
          INFO_PROVIDED, ESCALATED.
        excursion-specialist.json: 5 Resolution entries: INFO_PROVIDED,
          BOOKING_CANCELLED, ESCALATED.
        routing-judge.json: 10 RoutingScore entries, score 1–5, one-sentence rationale.
        booking-guardrail.json: 10 GuardrailVerdict entries; 7 allowed=true, 3
          allowed=false with violations "non-refundable-outside-window",
          "upgrade-above-entitlement", "insurance-prompt-bypassed".
        confirmation-gate.json: 8 ConfirmationRequest entries with plain-language
          summaries and a deadline 10 minutes from generation time.
    (b–e) Standard options from Lesson 25 (env var, env file, secrets URI, type-once).
- NEVER write the key value to any file. Record only the REFERENCE.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- (Lesson 1) AutonomousAgent is never silently downgraded. FlightSpecialist, HotelSpecialist,
  CarRentalSpecialist, ExcursionSpecialist all extend AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts: routeStep 20s, guardrailStep 20s, specialistStep /
  confirmStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on TravelRequest is Optional<T>.
- (Lesson 7) TravelTasks.java declares the RESOLVE Task<Resolution> constant. All
  specialists' definition().capability(TaskAcceptance.of(RESOLVE)...) reference it.
- (Lesson 8) Model names verified: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9236 in application.conf.
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
- The PiiSanitizer runs INSIDE a Consumer before any LLM call.
- The RouterEvalScorer Consumer reacts to RoutingDecided events and calls RoutingJudge
  via componentClient.forAgent(). It does NOT modify the workflow flow.
- The guardrail step happens BEFORE any booking mutation and BEFORE ConfirmationRequested.
  A blocked tool call never reaches the passenger — only blocked-draft surface for operator.
- The ConfirmationGate step happens AFTER guardrail clears and BEFORE ResolutionPublished.
  A REJECT terminates in PASSENGER_REJECTED with no mutation.
- No forbidden words in user-facing text.
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
