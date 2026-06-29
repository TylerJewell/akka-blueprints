# Data model — routing-classifier-pattern

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingMessage` | `messageId` | `String` | no | Server-assigned UUID. |
| | `channel` | `String` | no | `"email"` / `"chat"` / `"web-form"` / `"phone-transcript"`. |
| | `subject` | `String` | no | Raw subject line. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `RouteDecision` | `route` | `MessageRoute` | no | `GENERAL` / `REFUND` / `TECHNICAL` / `UNROUTABLE`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `RouteVerdict` | `approved` | `boolean` | no | `true` to proceed; `false` to block. |
| | `rejectionReason` | `String` | yes | Null when `approved=true`. Otherwise rubric token. |
| `Reply` | `replySubject` | `String` | no | `"Re: …"`. |
| | `replyBody` | `String` | no | 2–4 paragraphs. |
| | `action` | `ReplyAction` | no | `INFO_PROVIDED` / `REFUND_INITIATED` / `ARTICLE_LINKED` / `FOLLOW_UP_SCHEDULED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"general"` / `"refund"` / `"technical"`. |
| | `repliedAt` | `Instant` | no | When the specialist finished. |
| `ReplyVerdict` | `allowed` | `boolean` | no | `true` to publish; `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `Message` (entity state) | `messageId` | `String` | no | — |
| | `incoming` | `IncomingMessage` | no | Captured once at create. |
| | `routeDecision` | `Optional<RouteDecision>` | yes | Populated after `MessageClassified`. |
| | `routeVerdict` | `Optional<RouteVerdict>` | yes | Populated after route guardrail step. |
| | `reply` | `Optional<Reply>` | yes | Populated after `ReplyDrafted`. |
| | `replyVerdict` | `Optional<ReplyVerdict>` | yes | Populated after `ReplyVerdictAttached`. |
| | `abandonReason` | `Optional<String>` | yes | Populated after `MessageAbandoned`. |
| | `status` | `MessageStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Message` record is reused as the `MessageView` row type.

## Enums

`MessageRoute`: `GENERAL`, `REFUND`, `TECHNICAL`, `UNROUTABLE`.

`ReplyAction`: `INFO_PROVIDED`, `REFUND_INITIATED`, `ARTICLE_LINKED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`.

`MessageStatus`: `RECEIVED`, `CLASSIFIED`, `ROUTE_BLOCKED`, `ROUTED_GENERAL`, `ROUTED_REFUND`, `ROUTED_TECHNICAL`, `REPLY_DRAFTED`, `REPLY_BLOCKED`, `PUBLISHED`, `ABANDONED`.

## Events (`MessageEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `MessageRegistered` | `incoming` | `MessageIngestor.registerMessage` | → `RECEIVED` |
| `MessageClassified` | `routeDecision` | `RoutingWorkflow.classifyStep` | → `CLASSIFIED` |
| `MessageRouteBlocked` | `routeVerdict` | `RoutingWorkflow.validateRouteStep` (approved=false) | → `ROUTE_BLOCKED` (terminal) |
| `MessageRouted` | `route` | `RoutingWorkflow.validateRouteStep` (approved=true) | → `ROUTED_GENERAL` / `ROUTED_REFUND` / `ROUTED_TECHNICAL` |
| `ReplyDrafted` | `reply` | `RoutingWorkflow.generalStep` / `refundStep` / `technicalStep` | → `REPLY_DRAFTED` |
| `ReplyVerdictAttached` | `replyVerdict` | `RoutingWorkflow.screenStep` | (no status change) |
| `ReplyPublished` | — | `RoutingWorkflow.publishStep` (replyVerdict allowed) | → `PUBLISHED` (terminal) |
| `ReplyBlocked` | `violations` | `RoutingWorkflow.publishStep` (replyVerdict denied) | → `REPLY_BLOCKED` (terminal until unblock) |
| `MessageAbandoned` | `abandonReason` | `RoutingWorkflow.abandonStep` (route=UNROUTABLE or unrecoverable error) | → `ABANDONED` (terminal) |

A `REPLY_BLOCKED → PUBLISHED` transition is emitted via the `unblock` command, which writes a new `ReplyVerdictAttached` (with `allowed=true` and a `rubricVersion` of `"v1-operator-override"`) followed by `ReplyPublished`, and sets `finishedAt`.

Note: `ROUTE_BLOCKED` has no unblock command. It is a terminal state without a forward operator path — by design, a classifier decision that the route guardrail rejects should not be manually overridden to invoke a specialist.

## Events (`MessageQueue`)

| Event | Payload |
|---|---|
| `InboundMessageReceived` | `incoming` (the raw, pre-processing payload — audit log only) |

## View row

`MessageRow` is `Message` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without error. The view's single query is `SELECT * AS messages FROM message_view` with no `WHERE route` or `WHERE status` filter — callers (`RoutingEndpoint`) apply those client-side.
