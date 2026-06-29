# Data model — routing-classifier

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundTicket` | `caseId` | `String` | no | Server-assigned UUID. |
| | `customerId` | `String` | no | Opaque customer identifier from the inbound channel. |
| | `channel` | `String` | no | `"email"` / `"chat"` / `"web-form"` / `"phone-callback"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedTicket` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","card-number","account-id"]`. |
| `ClassificationResult` | `category` | `CaseCategory` | no | `BILLING` / `TECHNICAL` / `ACCOUNT` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `HandlerReply` | `replySubject` | `String` | no | `"Re: …"`. |
| | `replyBody` | `String` | no | 2–5 paragraphs. |
| | `action` | `ReplyAction` | no | `REFUND_INITIATED` / `ARTICLE_LINKED` / `FOLLOW_UP_SCHEDULED` / `INFO_PROVIDED` / `ACCOUNT_UPDATED` / `ESCALATED`. |
| | `handlerTag` | `String` | no | `"billing"` / `"technical"` / `"account"`. |
| | `repliedAt` | `Instant` | no | When the handler finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `Case` (entity state) | `caseId` | `String` | no | — |
| | `incoming` | `InboundTicket` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedTicket>` | yes | Populated after `CaseSanitized`. |
| | `classification` | `Optional<ClassificationResult>` | yes | Populated after `CaseClassified`. |
| | `reply` | `Optional<HandlerReply>` | yes | Populated after `ReplyDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `CaseEscalated`. |
| | `status` | `CaseStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CaseRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Case` record is reused as the `CaseView` row type.

## Enums

`CaseCategory`: `BILLING`, `TECHNICAL`, `ACCOUNT`, `UNCLEAR`.

`ReplyAction`: `REFUND_INITIATED`, `ARTICLE_LINKED`, `FOLLOW_UP_SCHEDULED`, `INFO_PROVIDED`, `ACCOUNT_UPDATED`, `ESCALATED`.

`CaseStatus`: `RECEIVED`, `SANITIZED`, `CLASSIFIED`, `ROUTED_BILLING`, `ROUTED_TECHNICAL`, `ROUTED_ACCOUNT`, `REPLY_DRAFTED`, `BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`CaseEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `CaseRegistered` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `CaseSanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `CaseClassified` | `classification` | `RoutingWorkflow.classifyStep` | → `CLASSIFIED` |
| `CaseRouted` | `category` | `RoutingWorkflow.routeStep` | → `ROUTED_BILLING`, `ROUTED_TECHNICAL`, or `ROUTED_ACCOUNT` |
| `ReplyDrafted` | `reply` | `RoutingWorkflow.billingStep` / `technicalStep` / `accountStep` | → `REPLY_DRAFTED` |
| `GuardrailVerdictAttached` | `guardrail` | `RoutingWorkflow.guardrailStep` | (no status change) |
| `ReplyPublished` | — | `RoutingWorkflow.publishStep` (guardrail allowed) | → `RESOLVED` (terminal) |
| `ReplyBlocked` | `violations` | `RoutingWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `CaseEscalated` | `escalationReason` | `RoutingWorkflow.escalateStep` (classification UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |

A `BLOCKED → RESOLVED` transition is triggered by the `unblock` command, which writes both `GuardrailVerdictAttached` (with `allowed=true` and an override marker in `rubricVersion`) and `ReplyPublished` in the same command.

## Events (`TicketQueue`)

| Event | Payload |
|---|---|
| `InboundTicketReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`CaseRow` is `Case` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS cases FROM case_view` with no `WHERE category` or `WHERE status` filter — callers (`RoutingEndpoint`) apply those client-side.
