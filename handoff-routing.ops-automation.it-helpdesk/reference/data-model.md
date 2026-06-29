# Data model — it-helpdesk

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingRequest` | `requestId` | `String` | no | Server-assigned UUID. |
| | `submitterId` | `String` | no | Opaque employee identifier from the inbound channel. |
| | `channel` | `String` | no | `"email"` / `"chat"` / `"portal"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body; may contain credentials. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered the request. |
| `SanitizedRequest` | `redactedSubject` | `String` | no | Credentials stripped. |
| | `redactedBody` | `String` | no | Credentials stripped. |
| | `secretCategoriesFound` | `List<String>` | no | e.g. `["api-key","password","token","pem-block"]`. |
| `ClassificationDecision` | `category` | `RequestCategory` | no | `ACCESS` / `INFRASTRUCTURE` / `SOFTWARE` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `ProposedTicket` | `title` | `String` | no | Title for the ticketing queue entry. |
| | `assigneeGroup` | `String` | no | `"access-team"` / `"infra-team"` / `"software-team"`. |
| | `priority` | `String` | no | `"P1"` / `"P2"` / `"P3"`. |
| | `body` | `String` | no | Full ticket description; must name an approver if P1. |
| `Resolution` | `responseSubject` | `String` | no | `"Re: …"`. |
| | `responseBody` | `String` | no | 2–4 paragraphs. |
| | `action` | `ResolutionAction` | no | `TICKET_FILED` / `SELF_SERVICE_RESOLVED` / `RUNBOOK_LINKED` / `FOLLOW_UP_SCHEDULED` / `ESCALATED`. |
| | `proposedTicket` | `Optional<ProposedTicket>` | yes | Present only when the specialist proposes a ticket write. |
| | `specialistTag` | `String` | no | `"access"` / `"infra"` / `"software"`. |
| | `resolvedAt` | `Instant` | no | When the specialist finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to file, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Request` (entity state) | `requestId` | `String` | no | — |
| | `incoming` | `IncomingRequest` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedRequest>` | yes | Populated after `RequestSanitized`. |
| | `classification` | `Optional<ClassificationDecision>` | yes | Populated after `RequestClassified`. |
| | `resolution` | `Optional<Resolution>` | yes | Populated after `ResolutionDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `RequestEscalated`. |
| | `status` | `RequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RequestRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Request` record is reused as the `RequestView` row type.

## Enums

`RequestCategory`: `ACCESS`, `INFRASTRUCTURE`, `SOFTWARE`, `UNCLEAR`.

`ResolutionAction`: `TICKET_FILED`, `SELF_SERVICE_RESOLVED`, `RUNBOOK_LINKED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`.

`RequestStatus`: `RECEIVED`, `SANITIZED`, `CLASSIFIED`, `ROUTED_ACCESS`, `ROUTED_INFRASTRUCTURE`, `ROUTED_SOFTWARE`, `RESOLUTION_DRAFTED`, `TICKET_BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`RequestEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RequestRegistered` | `incoming` | `SecretSanitizer.registerIncoming` | → `RECEIVED` |
| `RequestSanitized` | `sanitized` | `SecretSanitizer.attachSanitized` | → `SANITIZED` |
| `RequestClassified` | `classification` | `HelpdeskWorkflow.classifyStep` | → `CLASSIFIED` |
| `RequestRouted` | `category` | `HelpdeskWorkflow.routeStep` | → `ROUTED_ACCESS` / `ROUTED_INFRASTRUCTURE` / `ROUTED_SOFTWARE` |
| `ResolutionDrafted` | `resolution` | `HelpdeskWorkflow.accessStep` / `infraStep` / `softwareStep` | → `RESOLUTION_DRAFTED` |
| `GuardrailVerdictAttached` | `guardrail` | `HelpdeskWorkflow.guardrailStep` | (no status change) |
| `TicketFiled` | `proposedTicket` | `HelpdeskWorkflow.fileStep` (guardrail allowed) | (no status change; ticket reference recorded) |
| `TicketBlocked` | `violations` | `HelpdeskWorkflow.guardrailStep` (guardrail denied) | → `TICKET_BLOCKED` (terminal until unblock) |
| `ResolutionPublished` | — | `HelpdeskWorkflow.publishStep` | → `RESOLVED` (terminal) |
| `RequestEscalated` | `escalationReason` | `HelpdeskWorkflow.escalateStep` (UNCLEAR or error) | → `ESCALATED` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RoutingEvalScorer` Consumer | (no status change; populates `routingScore`) |

A `TICKET_BLOCKED → RESOLVED` transition is emitted via the `unblock` command, which writes `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ResolutionPublished` in the same command.

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `InboundRequestReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`RequestRow` is `Request` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an error. The view's single query is `SELECT * AS requests FROM request_view` with no `WHERE category` or `WHERE status` filter — callers (`HelpdeskEndpoint`) apply those client-side.
