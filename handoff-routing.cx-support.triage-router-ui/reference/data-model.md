# Data model — triage-router-ui

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SupportMessage` | `messageId` | `String` | no | Server-assigned UUID. |
| | `customerId` | `String` | no | Opaque customer identifier from the inbound channel. |
| | `channel` | `String` | no | `"email"` / `"chat"` / `"web-form"`. |
| | `subject` | `String` | no | Message subject line. |
| | `body` | `String` | no | Message body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `RouteDecision` | `category` | `MessageCategory` | no | `BILLING` / `PRODUCT` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `HandlerReply` | `replySubject` | `String` | no | `"Re: …"`. |
| | `replyBody` | `String` | no | 3–5 paragraphs. |
| | `action` | `ReplyAction` | no | `REFUND_INITIATED` / `ARTICLE_LINKED` / `FOLLOW_UP_SCHEDULED` / `INFO_PROVIDED` / `ESCALATED`. |
| | `handlerTag` | `String` | no | `"billing"` / `"product"`. |
| | `repliedAt` | `Instant` | no | When the handler finished. |
| `GuardrailResult` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `Message` (entity state) | `messageId` | `String` | no | — |
| | `incoming` | `SupportMessage` | no | Captured once at registration. |
| | `route` | `Optional<RouteDecision>` | yes | Populated after `MessageRouted`. |
| | `reply` | `Optional<HandlerReply>` | yes | Populated after `ReplyDrafted`. |
| | `guardrail` | `Optional<GuardrailResult>` | yes | Populated after `GuardrailResultAttached`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `MessageEscalated`. |
| | `status` | `MessageStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Message` record is reused as the `MessageView` row type.

## Enums

`MessageCategory`: `BILLING`, `PRODUCT`, `UNCLEAR`.

`ReplyAction`: `REFUND_INITIATED`, `ARTICLE_LINKED`, `FOLLOW_UP_SCHEDULED`, `INFO_PROVIDED`, `ESCALATED`.

`MessageStatus`: `RECEIVED`, `ROUTED_BILLING`, `ROUTED_PRODUCT`, `REPLY_DRAFTED`, `BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`MessageEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `MessageRegistered` | `incoming` | `MessageSimulator` / `TriageEndpoint` | → `RECEIVED` |
| `MessageRouted` | `route` | `TriageWorkflow.routeStep` | → `ROUTED_BILLING` or `ROUTED_PRODUCT` |
| `ReplyDrafted` | `reply` | `TriageWorkflow.billingStep` / `productStep` | → `REPLY_DRAFTED` |
| `GuardrailResultAttached` | `guardrail` | `TriageWorkflow.guardrailStep` | (no status change) |
| `ReplyPublished` | — | `TriageWorkflow.publishStep` (guardrail allowed) | → `RESOLVED` (terminal) |
| `ReplyBlocked` | `violations` | `TriageWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `MessageEscalated` | `escalationReason` | `TriageWorkflow.escalateStep` (UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |

A `BLOCKED → RESOLVED` transition is emitted via the `unblock` command, which writes both `GuardrailResultAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ReplyPublished` in the same command.

## Events (`MessageQueue`)

| Event | Payload |
|---|---|
| `InboundMessageReceived` | `incoming` (the raw payload — used as the audit log before any LLM call) |

## View row

`MessageRow` is `Message` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS messages FROM message_view` with no `WHERE category` or `WHERE status` filter — callers (`TriageEndpoint`) apply those client-side.
