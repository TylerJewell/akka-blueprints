# Data model — inbox-classifier

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundMessage` | `messageId` | `String` | no | Server-assigned UUID. |
| | `senderId` | `String` | no | Opaque sender identifier from the inbound channel. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `channel` | `String` | no | `"gmail"` / `"imap"` / `"simulated"`. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedMessage` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","national-id","account-number"]`. |
| `ClassificationDecision` | `label` | `MessageLabel` | no | `URGENT` / `IMPORTANT` / `INFO` / `SPAM`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `RoutingDecision` | `action` | `RoutingAction` | no | `FLAG_URGENT` / `MOVE_TO_FOLDER` / `MARK_READ` / `MOVE_TO_SPAM` / `FORWARD_TO_HUMAN` / `DELETE`. |
| | `targetFolder` | `Optional<String>` | yes | Populated when `action = MOVE_TO_FOLDER`. |
| | `forwardAddress` | `Optional<String>` | yes | Populated when `action = FORWARD_TO_HUMAN`. |
| | `reason` | `String` | no | One short sentence. |
| | `decidedAt` | `Instant` | no | When the routing agent finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to execute, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `ClassificationScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Message` (entity state) | `messageId` | `String` | no | — |
| | `incoming` | `InboundMessage` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedMessage>` | yes | Populated after `MessageSanitized`. |
| | `classification` | `Optional<ClassificationDecision>` | yes | Populated after `MessageClassified`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `RoutingDecisionRecorded`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `classificationScore` | `Optional<ClassificationScore>` | yes | Populated after `ClassificationScored`. |
| | `blockReason` | `Optional<String>` | yes | Populated after `ActionBlocked` or `MessageEscalated`. |
| | `status` | `MessageStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Message` record is reused as the `MessageView` row type.

## Enums

`MessageLabel`: `URGENT`, `IMPORTANT`, `INFO`, `SPAM`.

`RoutingAction`: `FLAG_URGENT`, `MOVE_TO_FOLDER`, `MARK_READ`, `MOVE_TO_SPAM`, `FORWARD_TO_HUMAN`, `DELETE`.

`MessageStatus`: `RECEIVED`, `SANITIZED`, `CLASSIFIED`, `ROUTING_DECIDED`, `GUARDRAIL_PENDING`, `ACTIONED`, `BLOCKED`, `ESCALATED`.

## Events (`MessageEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `MessageRegistered` | `incoming` | `BodySanitizer.registerIncoming` | → `RECEIVED` |
| `MessageSanitized` | `sanitized` | `BodySanitizer.attachSanitized` | → `SANITIZED` |
| `MessageClassified` | `classification` | `RoutingWorkflow.classifyStep` | → `CLASSIFIED` |
| `RoutingDecisionRecorded` | `routing` | `RoutingWorkflow.routeStep` | → `ROUTING_DECIDED` |
| `GuardrailVerdictAttached` | `guardrail` | `RoutingWorkflow.guardrailStep` | → `GUARDRAIL_PENDING` |
| `ActionExecuted` | `routing.action` | `RoutingWorkflow.executeStep` (guardrail allowed or non-destructive) | → `ACTIONED` (terminal) |
| `ActionBlocked` | `violations` | `RoutingWorkflow.executeStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `MessageEscalated` | `blockReason` | `RoutingWorkflow` error handler | → `ESCALATED` (terminal) |
| `ClassificationScored` | `score, rationale, scoredAt` | `ClassificationEvalScorer` Consumer | (no status change; populates `classificationScore`) |

A `BLOCKED → ACTIONED` transition is emitted via the `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ActionExecuted` in the same command.

## Events (`InboxQueue`)

| Event | Payload |
|---|---|
| `InboundMessageReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`MessageRow` is `Message` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS messages FROM message_view` with no `WHERE label` or `WHERE status` filter — callers (`InboxEndpoint`) apply those client-side.
