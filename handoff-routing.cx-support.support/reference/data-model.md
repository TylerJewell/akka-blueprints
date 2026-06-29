# Data model — support

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingRequest` | `ticketId` | `String` | no | Server-assigned uuid. |
| | `customerId` | `String` | no | Opaque customer identifier from the inbound channel. |
| | `channel` | `String` | no | `"email"` / `"chat"` / `"web-form"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedRequest` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","card-number","order-id","account-id"]`. |
| `TriageDecision` | `category` | `TicketCategory` | no | `BILLING` / `TECHNICAL` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `Resolution` | `responseSubject` | `String` | no | `"Re: …"`. |
| | `responseBody` | `String` | no | 3–5 paragraphs. |
| | `action` | `ResolutionAction` | no | `REFUND_INITIATED` / `ARTICLE_LINKED` / `FOLLOW_UP_SCHEDULED` / `INFO_PROVIDED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"billing"` / `"technical"`. |
| | `resolvedAt` | `Instant` | no | When the specialist finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `TriageScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Ticket` (entity state) | `ticketId` | `String` | no | — |
| | `incoming` | `IncomingRequest` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedRequest>` | yes | Populated after `TicketSanitized`. |
| | `triage` | `Optional<TriageDecision>` | yes | Populated after `TriageDecided`. |
| | `resolution` | `Optional<Resolution>` | yes | Populated after `ResolutionDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `triageScore` | `Optional<TriageScore>` | yes | Populated after `TriageScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `TicketEscalated`. |
| | `status` | `TicketStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TicketRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Ticket` record is reused as the `TicketView` row type.

## Enums

`TicketCategory`: `BILLING`, `TECHNICAL`, `UNCLEAR`.

`ResolutionAction`: `REFUND_INITIATED`, `ARTICLE_LINKED`, `FOLLOW_UP_SCHEDULED`, `INFO_PROVIDED`, `ESCALATED`.

`TicketStatus`: `RECEIVED`, `SANITIZED`, `TRIAGED`, `ROUTED_BILLING`, `ROUTED_TECHNICAL`, `RESOLUTION_DRAFTED`, `BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`TicketEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `TicketRegistered` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `TicketSanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `TriageDecided` | `triage` | `SupportWorkflow.triageStep` | → `TRIAGED` |
| `TicketRouted` | `category` | `SupportWorkflow.routeStep` | → `ROUTED_BILLING` or `ROUTED_TECHNICAL` |
| `ResolutionDrafted` | `resolution` | `SupportWorkflow.billingStep` / `technicalStep` | → `RESOLUTION_DRAFTED` |
| `GuardrailVerdictAttached` | `guardrail` | `SupportWorkflow.guardrailStep` | (no status change) |
| `ResolutionPublished` | — | `SupportWorkflow.publishStep` (guardrail allowed) | → `RESOLVED` (terminal) |
| `ResolutionBlocked` | `violations` | `SupportWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `TicketEscalated` | `escalationReason` | `SupportWorkflow.escalateStep` (triage UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |
| `TriageScored` | `score, rationale, scoredAt` | `TriageEvalScorer` Consumer | (no status change; populates `triageScore`) |

A `BLOCKED → RESOLVED` transition is emitted via `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ResolutionPublished` in the same command.

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `InboundRequestReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`TicketRow` is `Ticket` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS tickets FROM ticket_view` with no `WHERE category` or `WHERE status` filter — callers (`SupportEndpoint`) apply those client-side.
