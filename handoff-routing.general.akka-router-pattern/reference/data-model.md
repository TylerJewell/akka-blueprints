# Data model — akka-router-pattern

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskRequest` | `requestId` | `String` | no | Server-assigned UUID. |
| | `requesterId` | `String` | no | Opaque requester identifier. |
| | `channel` | `String` | no | `"api"` / `"web-form"` / `"cli"`. |
| | `title` | `String` | no | Short title for the task. |
| | `body` | `String` | no | Full task description. |
| | `receivedAt` | `Instant` | no | When the request was received. |
| `ClassificationDecision` | `domain` | `TaskDomain` | no | `CONTENT` / `CODE` / `DATA` / `UNKNOWN`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `GuardrailVerdict` | `verdict` | `GuardrailOutcome` | no | `ALLOWED` / `DENIED`. |
| | `violations` | `List<String>` | no | Empty when `ALLOWED`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `TaskResult` | `resultTitle` | `String` | no | Short descriptive label for the output. |
| | `resultBody` | `String` | no | Full output text. |
| | `status` | `TaskResultStatus` | no | `COMPLETED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"content"` / `"code"` / `"data"`. |
| | `completedAt` | `Instant` | no | When the specialist finished. |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Request` (entity state) | `requestId` | `String` | no | — |
| | `incoming` | `TaskRequest` | no | Captured once at create. |
| | `classification` | `Optional<ClassificationDecision>` | yes | Populated after `ClassificationDecided`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `result` | `Optional<TaskResult>` | yes | Populated after `ResultPublished`. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `unroutableReason` | `Optional<String>` | yes | Populated after `RequestUnroutable`. |
| | `status` | `RequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RequestRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record.

## Enums

`TaskDomain`: `CONTENT`, `CODE`, `DATA`, `UNKNOWN`.

`GuardrailOutcome`: `ALLOWED`, `DENIED`.

`TaskResultStatus`: `COMPLETED`, `ESCALATED`.

`RequestStatus`: `RECEIVED`, `CLASSIFIED`, `GUARDRAIL_PASSED`, `ROUTED_CONTENT`, `ROUTED_CODE`, `ROUTED_DATA`, `EXECUTING`, `BLOCKED`, `COMPLETED`, `UNROUTABLE`.

## Events (`RequestEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RequestRegistered` | `incoming` | `QueueConsumer.registerRequest` | → `RECEIVED` |
| `ClassificationDecided` | `classification` | `RouterWorkflow.classifyStep` | → `CLASSIFIED` |
| `GuardrailVerdictAttached` | `guardrail` | `RouterWorkflow.guardrailStep` | → `GUARDRAIL_PASSED` (on ALLOWED) |
| `RequestBlocked` | `violations` | `RouterWorkflow.guardrailStep` (on DENIED) | → `BLOCKED` (terminal until unblock) |
| `RequestRouted` | `domain` | `RouterWorkflow.routeStep` | → `ROUTED_CONTENT` / `ROUTED_CODE` / `ROUTED_DATA` |
| `ExecutionStarted` | `specialistTag` | `RouterWorkflow.contentStep` / `codeStep` / `dataStep` | → `EXECUTING` |
| `ResultPublished` | `result` | `RouterWorkflow.publishStep` | → `COMPLETED` (terminal) |
| `RequestUnroutable` | `unroutableReason` | `RouterWorkflow.routeStep` (domain=UNKNOWN or unrecoverable error) | → `UNROUTABLE` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RoutingEvalScorer` Consumer | (no status change; populates `routingScore`) |

A `BLOCKED → COMPLETED` transition is emitted via `unblock` command, which writes both `GuardrailVerdictAttached` (`verdict=ALLOWED` with an override marker in `rubricVersion`) and `ResultPublished` in the same command.

## Events (`TaskQueue`)

| Event | Payload |
|---|---|
| `InboundTaskReceived` | `incoming` (the raw pre-classification payload — used as the audit log) |

## View row

`RequestRow` is `Request` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS requests FROM request_view` with no `WHERE domain` or `WHERE status` filter — callers (`RouterEndpoint`) apply those client-side.
