# Data model — travel-support-router

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingRequest` | `requestId` | `String` | no | Server-assigned UUID. |
| | `passengerId` | `String` | no | Opaque passenger identifier from the inbound channel. |
| | `channel` | `String` | no | `"web-chat"` / `"app"` / `"phone-transcript"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedRequest` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["passport-number","booking-reference","payment-card"]`. |
| `RoutingDecision` | `category` | `TravelCategory` | no | `FLIGHTS` / `HOTELS` / `CAR_RENTAL` / `EXCURSIONS` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `Resolution` | `responseSubject` | `String` | no | `"Re: …"`. |
| | `responseBody` | `String` | no | 3–5 paragraphs. |
| | `action` | `ResolutionAction` | no | See enum below. |
| | `specialistTag` | `String` | no | `"flights"` / `"hotels"` / `"car-rental"` / `"excursions"`. |
| | `bookingRef` | `Optional<String>` | yes | Extracted booking reference when present. |
| | `resolvedAt` | `Instant` | no | When the specialist finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to proceed, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `ConfirmationRequest` | `summary` | `String` | no | Plain-language one-sentence description of the proposed change. |
| | `deadline` | `Instant` | no | When the confirmation window closes (10 min from gate call). |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `TravelRequest` (entity state) | `requestId` | `String` | no | — |
| | `incoming` | `IncomingRequest` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedRequest>` | yes | Populated after `RequestSanitized`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `RoutingDecided`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `confirmation` | `Optional<ConfirmationRequest>` | yes | Populated after `ConfirmationRequested`. |
| | `confirmationOutcome` | `Optional<ConfirmationOutcome>` | yes | `CONFIRM` or `REJECT`, populated after passenger response. |
| | `resolution` | `Optional<Resolution>` | yes | Populated after `ResolutionDrafted`. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `RequestEscalated`. |
| | `status` | `RequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RequestRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `TravelRequest` record is reused as the `RequestView` row type.

## Enums

`TravelCategory`: `FLIGHTS`, `HOTELS`, `CAR_RENTAL`, `EXCURSIONS`, `UNCLEAR`.

`ResolutionAction`: `BOOKING_CHANGED`, `BOOKING_CANCELLED`, `BOOKING_CONFIRMED`, `INFO_PROVIDED`, `ARTICLE_LINKED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`.

`ConfirmationOutcome`: `CONFIRM`, `REJECT`.

`RequestStatus`: `RECEIVED`, `SANITIZED`, `ROUTED`, `ROUTED_FLIGHTS`, `ROUTED_HOTELS`, `ROUTED_CAR_RENTAL`, `ROUTED_EXCURSIONS`, `GUARDRAIL_CLEARED`, `AWAITING_CONFIRMATION`, `RESOLUTION_DRAFTED`, `BLOCKED`, `RESOLVED`, `PASSENGER_REJECTED`, `ESCALATED`.

## Events (`RequestEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RequestRegistered` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `RequestSanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `RoutingDecided` | `routing` | `TravelWorkflow.routeStep` | → `ROUTED` |
| `RequestRouted` | `category` | `TravelWorkflow.routeStep` | → `ROUTED_FLIGHTS` / `ROUTED_HOTELS` / `ROUTED_CAR_RENTAL` / `ROUTED_EXCURSIONS` |
| `GuardrailVerdictAttached` | `guardrail` | `TravelWorkflow.guardrailStep` | → `GUARDRAIL_CLEARED` (on allowed) |
| `ResolutionBlocked` | `violations` | `TravelWorkflow.guardrailStep` (denied) | → `BLOCKED` (terminal until unblock) |
| `ConfirmationRequested` | `confirmation` | `TravelWorkflow.confirmStep` | → `AWAITING_CONFIRMATION` (paused) |
| `PassengerConfirmed` | `passengerId` | `TravelEndpoint.confirm(CONFIRM)` | → `RESOLUTION_DRAFTED` (workflow resumes) |
| `PassengerRejected` | `passengerId` | `TravelEndpoint.confirm(REJECT)` | → `PASSENGER_REJECTED` (terminal) |
| `ResolutionDrafted` | `resolution` | `TravelWorkflow.publishStep` | → `RESOLUTION_DRAFTED` |
| `ResolutionPublished` | — | `TravelWorkflow.publishStep` (after confirm) | → `RESOLVED` (terminal) |
| `RequestEscalated` | `escalationReason` | `TravelWorkflow.escalateStep` (UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RouterEvalScorer` Consumer | (no status change; populates `routingScore`) |

A `BLOCKED → RESOLVED` transition is emitted via the `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ResolutionPublished` in the same command.

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `InboundRequestReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`TravelRequest` is used directly as the `RequestView` row type. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record. The view's single query is `SELECT * AS requests FROM request_view` with no `WHERE category` or `WHERE status` filter — callers (`TravelEndpoint`) apply those client-side.
