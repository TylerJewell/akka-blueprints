# Data model — support-multi-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundCase` | `caseId` | `String` | no | Server-assigned uuid. |
| | `customerId` | `String` | no | Opaque customer identifier from the inbound channel. |
| | `channel` | `String` | no | `"email"` / `"chat"` / `"web-form"` / `"phone"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedCase` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","card-number","account-id","customer-id"]`. |
| `RoutingDecision` | `category` | `CaseCategory` | no | `BILLING` / `TECHNICAL` / `ACCOUNT` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `Resolution` | `responseSubject` | `String` | no | `"Re: …"`. |
| | `responseBody` | `String` | no | 3–5 paragraphs. |
| | `action` | `ResolutionAction` | no | `REFUND_INITIATED` / `ARTICLE_LINKED` / `FOLLOW_UP_SCHEDULED` / `INFO_PROVIDED` / `ACCESS_RESTORED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"billing"` / `"technical"` / `"account"`. |
| | `resolvedAt` | `Instant` | no | When the specialist finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `Case` (entity state) | `caseId` | `String` | no | — |
| | `incoming` | `InboundCase` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedCase>` | yes | Populated after `CaseSanitized`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `CaseRouted`. |
| | `resolution` | `Optional<Resolution>` | yes | Populated after `ResolutionDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `CaseEscalated`. |
| | `status` | `CaseStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CaseRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Case` record is reused as the `CaseView` row type.

## Enums

`CaseCategory`: `BILLING`, `TECHNICAL`, `ACCOUNT`, `UNCLEAR`.

`ResolutionAction`: `REFUND_INITIATED`, `ARTICLE_LINKED`, `FOLLOW_UP_SCHEDULED`, `INFO_PROVIDED`, `ACCESS_RESTORED`, `ESCALATED`.

`CaseStatus`: `RECEIVED`, `SANITIZED`, `ROUTED_BILLING`, `ROUTED_TECHNICAL`, `ROUTED_ACCOUNT`, `RESOLUTION_DRAFTED`, `BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`CaseEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `CaseRegistered` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `CaseSanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `CaseRouted` | `category` | `SupportWorkflow.routeStep` | → `ROUTED_BILLING` or `ROUTED_TECHNICAL` or `ROUTED_ACCOUNT` |
| `ResolutionDrafted` | `resolution` | `SupportWorkflow.billingStep` / `technicalStep` / `accountStep` | → `RESOLUTION_DRAFTED` |
| `GuardrailVerdictAttached` | `guardrail` | `SupportWorkflow.guardrailStep` | (no status change) |
| `ResolutionPublished` | — | `SupportWorkflow.publishStep` (guardrail allowed) | → `RESOLVED` (terminal) |
| `ResolutionBlocked` | `violations` | `SupportWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `CaseEscalated` | `escalationReason` | `SupportWorkflow.escalateStep` (routing UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |

A `BLOCKED → RESOLVED` transition is triggered by the `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ResolutionPublished` in the same command.

## Events (`TicketQueue`)

| Event | Payload |
|---|---|
| `InboundCaseReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`CaseRow` is `Case` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS cases FROM case_view` with no `WHERE category` or `WHERE status` filter — callers (`SupportEndpoint`) apply those client-side.
