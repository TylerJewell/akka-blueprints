# Data model — airline-triage-router

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PassengerRequest` | `requestId` | `String` | no | Server-assigned UUID. |
| | `passengerId` | `String` | no | Opaque passenger identifier from the inbound channel. |
| | `channel` | `String` | no | `"web"` / `"mobile"` / `"phone-ivr"` / `"kiosk"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedPassengerRequest` | `redactedSubject` | `String` | no | PNR/PII redacted. |
| | `redactedBody` | `String` | no | PNR/PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["pnr","email","passport","frequent-flyer"]`. |
| `IntentDecision` | `intent` | `PassengerIntent` | no | `BOOKING` / `CHANGE` / `BAGGAGE` / `STATUS` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `AirlineResolution` | `responseSubject` | `String` | no | `"Re: …"`. |
| | `responseBody` | `String` | no | 2–5 paragraphs. |
| | `outcome` | `ResolutionOutcome` | no | See enum. |
| | `specialistTag` | `String` | no | `"booking"` / `"change"` / `"baggage"` / `"status"`. |
| | `resolvedAt` | `Instant` | no | When the specialist finished. |
| `ToolCallVerdict` | `allowed` | `boolean` | no | `true` to permit the tool call, `false` to deny. |
| | `denialReason` | `String` | yes | `null` when `allowed=true`. Policy-token string otherwise. |
| `ResponseVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Rubric-token list otherwise. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `FlightRequest` (entity state) | `requestId` | `String` | no | — |
| | `incoming` | `PassengerRequest` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedPassengerRequest>` | yes | Populated after `RequestSanitized`. |
| | `intent` | `Optional<IntentDecision>` | yes | Populated after `IntentClassified`. |
| | `resolution` | `Optional<AirlineResolution>` | yes | Populated after `ResolutionDrafted`. |
| | `toolCallVerdict` | `Optional<ToolCallVerdict>` | yes | Populated after `ToolCallVerdictAttached`. |
| | `responseVerdict` | `Optional<ResponseVerdict>` | yes | Populated after `ResponseVerdictAttached`. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `unresolvedReason` | `Optional<String>` | yes | Populated after `RequestUnresolved`. |
| | `status` | `FlightRequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RequestRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `FlightRequest` record is reused as the `FlightRequestView` row type.

## Enums

`PassengerIntent`: `BOOKING`, `CHANGE`, `BAGGAGE`, `STATUS`, `UNCLEAR`.

`ResolutionOutcome`: `BOOKING_CONFIRMED`, `CHANGE_APPLIED`, `BAGGAGE_CLAIM_FILED`, `STATUS_PROVIDED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`.

`FlightRequestStatus`: `RECEIVED`, `SANITIZED`, `CLASSIFIED`, `ROUTED_BOOKING`, `ROUTED_CHANGE`, `ROUTED_BAGGAGE`, `ROUTED_STATUS`, `RESOLUTION_DRAFTED`, `BLOCKED`, `RESOLVED`, `UNRESOLVED`.

## Events (`FlightRequestEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RequestRegistered` | `incoming` | `PnrSanitizer.registerIncoming` | → `RECEIVED` |
| `RequestSanitized` | `sanitized` | `PnrSanitizer.attachSanitized` | → `SANITIZED` |
| `IntentClassified` | `intent` | `FlightRequestWorkflow.classifyStep` | → `CLASSIFIED` |
| `RequestRouted` | `intent` | `FlightRequestWorkflow.routeStep` | → `ROUTED_BOOKING` / `ROUTED_CHANGE` / `ROUTED_BAGGAGE` / `ROUTED_STATUS` |
| `ResolutionDrafted` | `resolution` | `FlightRequestWorkflow.bookingStep` etc. | → `RESOLUTION_DRAFTED` |
| `ToolCallVerdictAttached` | `toolCallVerdict` | `FlightRequestWorkflow` (specialist inline) | (no status change, or → `BLOCKED` on denial) |
| `ResponseVerdictAttached` | `responseVerdict` | `FlightRequestWorkflow.responseGuardrailStep` | (no status change) |
| `ResponsePublished` | — | `FlightRequestWorkflow.publishStep` (verdict allowed) | → `RESOLVED` (terminal) |
| `ResponseBlocked` | `violations` | `FlightRequestWorkflow.publishStep` (verdict denied) | → `BLOCKED` (terminal until unblock) |
| `RequestUnresolved` | `unresolvedReason` | `FlightRequestWorkflow.unresolvedStep` (intent UNCLEAR or unrecoverable error) | → `UNRESOLVED` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RoutingEvalScorer` Consumer | (no status change; populates `routingScore`) |

A `BLOCKED → RESOLVED` transition is emitted via the `unblock` command, which writes both `ResponseVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ResponsePublished` in the same command.

## Events (`PassengerRequestQueue`)

| Event | Payload |
|---|---|
| `InboundPassengerRequestReceived` | `incoming` (the raw, pre-sanitization payload — the audit log) |

## View row

`FlightRequestRow` is `FlightRequest` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without error. The view's single query is `SELECT * AS requests FROM flight_request_view` with no `WHERE intent` or `WHERE status` filter — callers (`AirlineEndpoint`) apply those client-side.
