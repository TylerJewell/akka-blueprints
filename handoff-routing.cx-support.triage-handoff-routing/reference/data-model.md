# Data model — triage-handoff-routing

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundTurn` | `turnId` | `String` | no | Server-assigned UUID. |
| | `sessionId` | `String` | no | Opaque session identifier from the inbound channel. |
| | `channel` | `String` | no | `"chat"` / `"email"` / `"web-widget"`. |
| | `rawText` | `String` | no | Raw text as received; *not yet normalised*. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `NormalisedTurn` | `normalisedText` | `String` | no | Lower-cased, whitespace-collapsed text. |
| | `languageCode` | `String` | no | BCP-47 language code, e.g. `"en"`, `"fr"`. |
| | `containsSensitiveData` | `boolean` | no | `true` if the classifier flagged PII-adjacent tokens. |
| `TriageDecision` | `intent` | `IntentCategory` | no | `ACCOUNT` / `PRODUCT` / `RETURNS` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `RoutingVerdict` | `allowed` | `boolean` | no | `true` to proceed to specialist, `false` to block. |
| | `reason` | `String` | no | One short sentence — what passed or failed. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `Reply` | `responseText` | `String` | no | 2–4 paragraphs. |
| | `action` | `ReplyAction` | no | `INFORMATION_PROVIDED` / `ACCOUNT_UPDATED` / `RETURN_INITIATED` / `FOLLOW_UP_SCHEDULED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"account"` / `"product"` / `"returns"`. |
| | `repliedAt` | `Instant` | no | When the specialist finished. |
| `ResponseVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `Conversation` (entity state) | `turnId` | `String` | no | — |
| | `inbound` | `InboundTurn` | no | Captured once at create. |
| | `normalised` | `Optional<NormalisedTurn>` | yes | Populated after `TurnNormalised`. |
| | `triage` | `Optional<TriageDecision>` | yes | Populated after `TurnTriaged`. |
| | `routing` | `Optional<RoutingVerdict>` | yes | Populated after `RoutingBlocked` or `TurnRouted`. |
| | `reply` | `Optional<Reply>` | yes | Populated after `ReplyDrafted`. |
| | `responseVerdict` | `Optional<ResponseVerdict>` | yes | Populated after `ResponseVerdictAttached`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `TurnEscalated`. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TurnRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Conversation` record is reused as the `ConversationView` row type.

## Enums

`IntentCategory`: `ACCOUNT`, `PRODUCT`, `RETURNS`, `UNCLEAR`.

`ReplyAction`: `INFORMATION_PROVIDED`, `ACCOUNT_UPDATED`, `RETURN_INITIATED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`.

`ConversationStatus`: `RECEIVED`, `NORMALISED`, `TRIAGED`, `ROUTING_BLOCKED`, `ROUTED_ACCOUNT`, `ROUTED_PRODUCT`, `ROUTED_RETURNS`, `REPLY_DRAFTED`, `RESPONSE_BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`ConversationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `TurnRegistered` | `inbound` | `IntentClassifier.registerTurn` | → `RECEIVED` |
| `TurnNormalised` | `normalised` | `IntentClassifier.attachNormalised` | → `NORMALISED` |
| `TurnTriaged` | `triage` | `HandoffWorkflow.triageStep` | → `TRIAGED` |
| `RoutingBlocked` | `routing (allowed=false)` | `HandoffWorkflow.routingCheckStep` (guardrail denied) | → `ROUTING_BLOCKED` (terminal until review) |
| `TurnRouted` | `intent` | `HandoffWorkflow.routingCheckStep` (guardrail allowed) | → `ROUTED_ACCOUNT` / `ROUTED_PRODUCT` / `ROUTED_RETURNS` |
| `ReplyDrafted` | `reply` | `HandoffWorkflow.accountStep` / `productStep` / `returnStep` | → `REPLY_DRAFTED` |
| `ResponseVerdictAttached` | `responseVerdict` | `HandoffWorkflow.responseCheckStep` | (no status change — verdict attached for audit) |
| `ReplyPublished` | — | `HandoffWorkflow.publishStep` (response guardrail allowed) | → `RESOLVED` (terminal) |
| `ReplyBlocked` | `violations` | `HandoffWorkflow.responseCheckStep` (response guardrail denied) | → `RESPONSE_BLOCKED` (terminal until review) |
| `TurnEscalated` | `escalationReason` | `HandoffWorkflow.escalateStep` (intent UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |

A `ROUTING_BLOCKED → RESOLVED` or `RESPONSE_BLOCKED → RESOLVED` transition is emitted via the `review` command with `publish=true`, which writes a `ReplyPublished` event with an operator-override marker in the audit note.

A `ROUTING_BLOCKED → ESCALATED` or `RESPONSE_BLOCKED → ESCALATED` transition is emitted via the `review` command with `publish=false`, which writes a `TurnEscalated` event with the reviewer's note.

## Events (`ConversationQueue`)

| Event | Payload |
|---|---|
| `InboundTurnReceived` | `inbound` (the raw, pre-normalisation payload — used as the audit log) |

## View row

`ConversationRow` is `Conversation` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS turns FROM conversation_view` with no `WHERE intent` or `WHERE status` filter — callers (`HandoffEndpoint`) apply those client-side.
