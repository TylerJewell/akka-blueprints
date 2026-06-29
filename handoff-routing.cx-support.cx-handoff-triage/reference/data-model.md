# Data model — cx-handoff-triage

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundMessage` | `conversationId` | `String` | no | Server-assigned UUID. |
| | `channel` | `String` | no | `"chat"` / `"email"` / `"voice-transcript"`. |
| | `openingMessage` | `String` | no | Raw opening message; *not yet sanitized*. |
| | `customerHandle` | `String` | no | Opaque customer identifier from the channel. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedMessage` | `redactedText` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","order-id"]`. |
| `TriageDecision` | `category` | `ConversationCategory` | no | `SALES` / `ISSUES_REPAIRS` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `SpecialistResponse` | `responseText` | `String` | no | 3–5 paragraphs. |
| | `action` | `ResponseAction` | no | `ORDER_PLACED` / `REFUND_INITIATED` / `REPLACEMENT_ARRANGED` / `INFO_PROVIDED` / `FOLLOW_UP_SCHEDULED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"sales"` / `"issues-repairs"`. |
| | `resolvedAt` | `Instant` | no | When the specialist finished. |
| `ToolCallRequest` | `toolName` | `String` | no | `"processRefund"` / `"placeOrder"` / `"scheduleReplacement"`. |
| | `conversationId` | `String` | no | Link back to the parent conversation. |
| | `parameters` | `Map<String, Object>` | no | Tool-specific parameter map. |
| `ToolCallVerdict` | `allowed` | `boolean` | no | `true` to execute, `false` to cancel. |
| | `reason` | `String` | no | Empty when `allowed=true`; violation token when denied. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `Conversation` (entity state) | `conversationId` | `String` | no | — |
| | `incoming` | `InboundMessage` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedMessage>` | yes | Populated after `ConversationSanitized`. |
| | `triage` | `Optional<TriageDecision>` | yes | Populated after `ConversationTriaged`. |
| | `response` | `Optional<SpecialistResponse>` | yes | Populated after `ResponseDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `ConversationEscalated`. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ConversationRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record.

## Enums

`ConversationCategory`: `SALES`, `ISSUES_REPAIRS`, `UNCLEAR`.

`ResponseAction`: `ORDER_PLACED`, `REFUND_INITIATED`, `REPLACEMENT_ARRANGED`, `INFO_PROVIDED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`.

`ConversationStatus`: `RECEIVED`, `SANITIZED`, `TRIAGED`, `ROUTED_SALES`, `ROUTED_ISSUES_REPAIRS`, `RESPONSE_DRAFTED`, `TOOL_CALL_BLOCKED`, `BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`ConversationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ConversationRegistered` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `ConversationSanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `ConversationTriaged` | `triage` | `ConversationWorkflow.triageStep` | → `TRIAGED` |
| `ConversationRouted` | `category` | `ConversationWorkflow.routeStep` | → `ROUTED_SALES` or `ROUTED_ISSUES_REPAIRS` |
| `ResponseDrafted` | `response` | `ConversationWorkflow.salesStep` / `issuesStep` | → `RESPONSE_DRAFTED` |
| `ToolCallBlocked` | `toolName, reason` | `ToolCallGuardrail` verdict `allowed=false` | → `TOOL_CALL_BLOCKED` (transient; specialist continues) |
| `GuardrailVerdictAttached` | `guardrail` | `ConversationWorkflow.guardrailStep` | (no status change) |
| `ResponsePublished` | — | `ConversationWorkflow.publishStep` (guardrail allowed) | → `RESOLVED` (terminal) |
| `ResponseBlocked` | `violations` | `ConversationWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `ConversationEscalated` | `escalationReason` | `ConversationWorkflow.escalateStep` (triage UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |

A `BLOCKED → RESOLVED` transition is emitted via the `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ResponsePublished` in the same command.

## Events (`ConversationQueue`)

| Event | Payload |
|---|---|
| `InboundMessageReceived` | `incoming` (the raw, pre-sanitization payload — audit log) |

## View row

`ConversationRow` mirrors `Conversation` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS conversations FROM conversation_view` with no `WHERE category` or `WHERE status` filter — callers (`ChatEndpoint`) apply those client-side.
