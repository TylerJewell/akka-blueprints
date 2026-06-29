# Data model — aws-ops-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `AwsAction` | `actionId` | `String` | no | UUID minted by the agent for this planned call. |
| | `awsService` | `String` | no | AWS service name (e.g., `"EC2"`, `"S3"`, `"Lambda"`). |
| | `apiCall` | `String` | no | AWS API operation name (e.g., `"DescribeInstances"`). |
| | `resourceArn` | `String` | no | Target ARN, prefix, or `"*"` for list-style reads. |
| | `kind` | `ActionKind` | no | `READ` or `MUTATING`. |
| | `rationale` | `String` | no | One-sentence agent justification. |
| `ConfirmationRequest` | `actionId` | `String` | no | Matches an `AwsAction.actionId`. |
| | `action` | `AwsAction` | no | The full planned action for the UI to render. |
| | `requestedAt` | `Instant` | no | When the workflow emitted `ConfirmationRequested`. |
| `ConfirmationDecision` | `actionId` | `String` | no | Matches the `ConfirmationRequest.actionId`. |
| | `outcome` | `ConfirmationOutcome` | no | `APPROVED` or `DECLINED`. |
| | `decidedBy` | `String` | no | User identifier. |
| | `decidedAt` | `Instant` | no | When the endpoint received the decision. |
| `ActionResult` | `actionId` | `String` | no | Matches the corresponding `AwsAction.actionId`. |
| | `action` | `AwsAction` | no | The action that was attempted. |
| | `outcome` | `ActionOutcome` | no | `COMPLETED`, `SKIPPED`, `BLOCKED`, or `FAILED`. |
| | `responseSnippet` | `String` | no | Truncated API response (≤ 512 chars) or rejection/skip reason. |
| | `durationMs` | `long` | no | Elapsed time for the MCP call (0 for SKIPPED/BLOCKED). |
| `OpsRequest` | `opsRequestId` | `String` | no | UUID minted by `OpsEndpoint`. |
| | `requestText` | `String` | no | Natural-language operations request. |
| | `context` | `String` | no | Optional environment tag or account alias (empty string if absent). |
| | `scope` | `RequestScope` | no | `READ_ONLY`, `MUTATING`, or `MIXED`. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `OperationReport` | `status` | `ReportStatus` | no | `COMPLETED`, `BLOCKED`, or `HALTED`. |
| | `summary` | `String` | no | 1–3 sentences. |
| | `actions` | `List<ActionResult>` | no | One entry per attempted action. |
| | `completedAt` | `Instant` | no | When the agent returned the report. |
| `OpsRequestState` (entity state) | `opsRequestId` | `String` | no | — |
| | `request` | `Optional<OpsRequest>` | yes | Populated after `RequestSubmitted`. |
| | `pendingConfirmations` | `List<ConfirmationRequest>` | no | Grows on `ConfirmationRequested`; shrinks on `ConfirmationReceived`. |
| | `decisions` | `List<ConfirmationDecision>` | no | Append-only log of all APPROVED/DECLINED decisions. |
| | `report` | `Optional<OperationReport>` | yes | Populated after `ReportRecorded`. |
| | `status` | `OpsRequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RequestSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `OpsRequestState` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ActionKind`: `READ`, `MUTATING`.
`ConfirmationOutcome`: `APPROVED`, `DECLINED`.
`ActionOutcome`: `COMPLETED`, `SKIPPED`, `BLOCKED`, `FAILED`.
`RequestScope`: `READ_ONLY`, `MUTATING`, `MIXED`.
`ReportStatus`: `COMPLETED`, `BLOCKED`, `HALTED`.
`OpsRequestStatus`: `SUBMITTED`, `PLANNING`, `AWAITING_CONFIRMATION`, `EXECUTING`, `COMPLETED`, `HALTED`, `FAILED`.

## Events (`OpsRequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RequestSubmitted` | `request` | → SUBMITTED |
| `PlanningStarted` | — | → PLANNING |
| `ConfirmationRequested` | `confirmationRequest` | → AWAITING_CONFIRMATION |
| `ConfirmationDelivered` | `actionId` | (no status change; bookkeeping) |
| `ConfirmationReceived` | `decision` | → EXECUTING (decision recorded) |
| `ExecutionStarted` | — | → EXECUTING (read-only fast path) |
| `ReportRecorded` | `report` | → COMPLETED (terminal happy) |
| `RequestHalted` | `haltedBy`, `haltedAt` | → HALTED (terminal) |
| `RequestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `OpsRequestState.initial("")` with `request = Optional.empty()`, `pendingConfirmations = List.of()`, `decisions = List.of()`, `report = Optional.empty()`, `finishedAt = Optional.empty()`, `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`OpsRequestRow` mirrors `OpsRequestState`. The row carries the full `pendingConfirmations` list (for the UI confirmation cards) and `decisions` list, but trims `report.actions[*].responseSnippet` to the first 128 chars in the view to keep row size manageable. Full snippets are available via `GET /api/ops-requests/{id}` which reads directly from the entity.

The view declares ONE query: `getAllRequests: SELECT * AS requests FROM ops_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`OpsTasks.java`)

```java
public final class OpsTasks {
  public static final Task<OperationReport> EXECUTE_OPS_REQUEST = Task
      .name("Execute AWS operations request")
      .description(
          "Parse the natural-language request, call AWS APIs one at a time via MCP tools, " +
          "pause for human confirmation before mutating calls, and return an OperationReport")
      .resultConformsTo(OperationReport.class);

  private OpsTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
