# Data model — multi-agent-handoff-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingTask` | `taskId` | `String` | no | Server-assigned uuid. |
| | `requesterId` | `String` | no | Opaque identifier of the requester. |
| | `title` | `String` | no | Short title; ≤ 150 characters. |
| | `description` | `String` | no | Full task description; 10–4000 characters. |
| | `preferredDomain` | `String` | no | `"data-analysis"` / `"content-writing"` / `"code-review"` / `"auto"`. |
| | `receivedAt` | `Instant` | no | When the queue received the request. |
| `AdmissionResult` | `admitted` | `boolean` | no | `true` when all checks pass. |
| | `rejectionReasons` | `List<String>` | no | Empty when `admitted=true`. |
| `RoutingDecision` | `domain` | `TaskDomain` | no | `DATA_ANALYSIS` / `CONTENT_WRITING` / `CODE_REVIEW` / `UNROUTABLE`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `TaskResult` | `headline` | `String` | no | One-sentence summary; ≤ 100 characters. |
| | `body` | `String` | no | Produced content; three to five paragraphs or JSON. |
| | `format` | `ResultFormat` | no | `MARKDOWN_REPORT` / `STRUCTURED_JSON` / `INLINE_COMMENT` / `PLAIN_TEXT`. |
| | `specialistTag` | `String` | no | `"data-analyst"` / `"content-writer"` / `"code-reviewer"`. |
| | `completedAt` | `Instant` | no | When the specialist finished. |
| `ToolCallRecord` | `toolName` | `String` | no | Name of the tool the specialist attempted. |
| | `argsDigest` | `String` | no | First 8 hex chars of SHA-256 of serialised args. |
| | `allowed` | `boolean` | no | `false` when blocked. |
| | `blockReason` | `Optional<String>` | yes | Populated when `allowed=false`. |
| | `calledAt` | `Instant` | no | When the call was intercepted. |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Task` (entity state) | `taskId` | `String` | no | — |
| | `incoming` | `IncomingTask` | no | Captured once at create. |
| | `admission` | `Optional<AdmissionResult>` | yes | Populated after `TaskAdmitted` or `TaskRejectedAtAdmission`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `RoutingDecided`. |
| | `result` | `Optional<TaskResult>` | yes | Populated after `ResultDrafted`. |
| | `blockedTool` | `Optional<ToolCallRecord>` | yes | Populated after `ToolCallBlocked`. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated after `TaskRejected` or `TaskRejectedAtAdmission`. |
| | `status` | `TaskStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TaskReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record.

## Enums

`TaskDomain`: `DATA_ANALYSIS`, `CONTENT_WRITING`, `CODE_REVIEW`, `UNROUTABLE`.

`ResultFormat`: `MARKDOWN_REPORT`, `STRUCTURED_JSON`, `INLINE_COMMENT`, `PLAIN_TEXT`.

`TaskStatus`: `RECEIVED`, `ADMITTED`, `ROUTED`, `ROUTED_DATA_ANALYSIS`, `ROUTED_CONTENT_WRITING`, `ROUTED_CODE_REVIEW`, `EXECUTING`, `TOOL_BLOCKED`, `VALIDATION_FAILED`, `COMPLETED`, `REJECTED`.

## Events (`TaskEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `TaskReceived` | `incoming` | `AdmissionGuardrail.registerIncoming` | → `RECEIVED` |
| `TaskAdmitted` | `admission` | `AdmissionGuardrail.recordAdmission(admitted=true)` | → `ADMITTED` |
| `TaskRejectedAtAdmission` | `rejectionReasons` | `AdmissionGuardrail.rejectAtAdmission` | → `REJECTED` (terminal) |
| `RoutingDecided` | `routing` | `HandoffWorkflow.routeStep` | → `ROUTED` |
| `TaskRouted` | `domain` | `HandoffWorkflow.branchStep` | → `ROUTED_DATA_ANALYSIS` / `ROUTED_CONTENT_WRITING` / `ROUTED_CODE_REVIEW` |
| `ToolCallBlocked` | `blockedTool` | `HandoffWorkflow.<specialist>Step` (tool interceptor) | → `TOOL_BLOCKED` (terminal) |
| `ResultDrafted` | `result` | `HandoffWorkflow.<specialist>Step` (on specialist return) | → `EXECUTING` |
| `ValidationFailed` | `message` | `HandoffWorkflow.validateStep` (check fails) | → `VALIDATION_FAILED` (terminal) |
| `ResultPublished` | — | `HandoffWorkflow.publishStep` (validation passes) | → `COMPLETED` (terminal) |
| `TaskRejected` | `rejectionReason` | `HandoffWorkflow.rejectStep` (domain = UNROUTABLE or unrecoverable error) | → `REJECTED` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RoutingEvalScorer` Consumer | (no status change; populates `routingScore`) |

## Events (`TaskQueue`)

| Event | Payload |
|---|---|
| `InboundTaskReceived` | `incoming` (the raw, pre-admission payload — used as the audit log) |

## View row

`TaskRow` is `Task` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS tasks FROM task_view` with no `WHERE domain` or `WHERE status` filter — callers (`WorkflowEndpoint`) apply those client-side.
