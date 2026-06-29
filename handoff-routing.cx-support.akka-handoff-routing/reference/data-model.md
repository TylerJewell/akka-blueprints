# Data model — akka-handoff-routing

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundTurn` | `conversationId` | `String` | no | Server-assigned uuid. |
| | `customerId` | `String` | no | Opaque customer identifier from the inbound channel. |
| | `channel` | `String` | no | `"chat"` / `"email"` / `"web-form"`. |
| | `userMessage` | `String` | no | Raw message text; *not yet filtered*. |
| | `priorTurns` | `List<String>` | no | Filtered summaries of earlier turns in the conversation. Empty list if first turn. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `FilteredContext` | `filteredMessage` | `String` | no | PII-redacted message text. |
| | `filteredPriorTurns` | `List<String>` | no | PII-redacted prior-turn summaries. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","card-number","account-id"]`. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to proceed, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Rubric-token list when rejected. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `RoutingDecision` | `category` | `RoutingCategory` | no | `BILLING` / `TECHNICAL` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `Reply` | `replyBody` | `String` | no | 2–4 paragraphs. |
| | `action` | `ReplyAction` | no | `REFUND_INITIATED` / `ARTICLE_LINKED` / `FOLLOW_UP_SCHEDULED` / `INFO_PROVIDED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"billing"` / `"technical"`. |
| | `repliedAt` | `Instant` | no | When the specialist finished. |
| `HandoffScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Conversation` (entity state) | `conversationId` | `String` | no | — |
| | `inbound` | `InboundTurn` | no | Captured once at create. |
| | `filtered` | `Optional<FilteredContext>` | yes | Populated after `ConversationFiltered`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `RoutingDecided`. |
| | `reply` | `Optional<Reply>` | yes | Populated after `ReplyDrafted`. |
| | `handoffScore` | `Optional<HandoffScore>` | yes | Populated after `HandoffScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `ConversationEscalated`. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ConversationRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Conversation` record is reused as the `ConversationView` row type.

## Enums

`RoutingCategory`: `BILLING`, `TECHNICAL`, `UNCLEAR`.

`ReplyAction`: `REFUND_INITIATED`, `ARTICLE_LINKED`, `FOLLOW_UP_SCHEDULED`, `INFO_PROVIDED`, `ESCALATED`.

`ConversationStatus`: `RECEIVED`, `FILTERED`, `GUARDRAIL_PASSED`, `ROUTED_BILLING`, `ROUTED_TECHNICAL`, `REPLY_READY`, `BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`ConversationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ConversationRegistered` | `inbound` | `MessageFilter.registerInbound` | → `RECEIVED` |
| `ConversationFiltered` | `filtered` | `MessageFilter.attachFiltered` | → `FILTERED` |
| `GuardrailVerdictAttached` | `guardrail` | `HandoffWorkflow.guardrailStep` | → `GUARDRAIL_PASSED` (if allowed) or stays `FILTERED` before `HandoffBlocked` |
| `RoutingDecided` | `routing` | `HandoffWorkflow.triageStep` | → `GUARDRAIL_PASSED` (no status change; routing is set) |
| `ConversationRouted` | `category` | `HandoffWorkflow.routeStep` | → `ROUTED_BILLING` or `ROUTED_TECHNICAL` |
| `ReplyDrafted` | `reply` | `HandoffWorkflow.billingStep` / `technicalStep` | → `REPLY_READY` |
| `ReplyPublished` | — | `HandoffWorkflow.publishStep` | → `RESOLVED` (terminal) |
| `HandoffBlocked` | `violations` | `HandoffWorkflow.guardrailStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `ConversationEscalated` | `escalationReason` | `HandoffWorkflow.escalateStep` (routing UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |
| `HandoffScored` | `score, rationale, scoredAt` | `HandoffEvalScorer` Consumer | (no status change; populates `handoffScore`) |

A `BLOCKED → RESOLVED` transition is triggered by the `unblock` command, which writes `GuardrailVerdictAttached` (allowed=true with an override marker in `rubricVersion`) and `ReplyPublished` in the same command.

Note: `HandoffScored` is only emitted for conversations that reach `RoutingDecided`. Conversations blocked by the guardrail before classification never emit `RoutingDecided` and therefore never emit `HandoffScored`.

## Events (`ConversationQueue`)

| Event | Payload |
|---|---|
| `InboundTurnReceived` | `inbound` (the raw, pre-filter payload — used as the audit log) |

## View row

`ConversationRow` is `Conversation` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS conversations FROM conversation_view` with no `WHERE category` or `WHERE status` filter — callers (`HandoffEndpoint`) apply those client-side.
