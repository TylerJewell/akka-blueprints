# Data model — autonomous-agent-pattern

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IssueRef` | `issueUrl` | `String` | no | Full GitHub issue URL. |
| | `repoOwner` | `String` | no | GitHub org or user. |
| | `repoName` | `String` | no | Repository name. |
| | `issueNumber` | `int` | no | Issue number parsed from URL. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When `AgentTaskEndpoint` received the request. |
| `IssueBody` | `title` | `String` | no | Issue title from GitHub API. |
| | `body` | `String` | no | Issue body text. |
| | `labels` | `List<String>` | no | Label names attached to the issue. |
| | `fetchedAt` | `String` | no | ISO-8601 timestamp of the fetch. |
| `ToolCall` | `toolCallId` | `String` | no | UUID minted per call. |
| | `toolName` | `String` | no | One of `{readFile, writeFile, runShell, searchCodebase}`. |
| | `parameters` | `Map<String, String>` | no | Tool parameters (e.g., `path`, `command`, `query`). |
| | `status` | `ToolCallStatus` | no | Enum value. |
| | `result` | `Optional<String>` | yes | Tool output (absent when BLOCKED or FAILED). |
| | `blockReason` | `Optional<String>` | yes | Guardrail rejection reason (present only when BLOCKED). |
| | `calledAt` | `Instant` | no | When the guardrail processed this call. |
| `AgentPlan` | `summary` | `String` | no | 2–3 sentence description. |
| | `filesToChange` | `List<String>` | no | Relative paths of files the fix will touch. |
| | `approach` | `String` | no | One paragraph on fix strategy. |
| | `plannedAt` | `Instant` | no | When the plan agent returned. |
| `PatchDiff` | `unifiedDiff` | `String` | no | Standard unified diff output. |
| | `filesModified` | `List<String>` | no | Files actually written. |
| | `linesAdded` | `int` | no | Total lines added. |
| | `linesRemoved` | `int` | no | Total lines removed. |
| `TaskOutcome` | `status` | `OutcomeStatus` | no | Enum value. |
| | `patch` | `Optional<PatchDiff>` | yes | Absent on `FAILED`. |
| | `confidenceScore` | `double` | no | 0.0–1.0. |
| | `toolTrace` | `List<ToolCall>` | no | All tool calls during execute phase. |
| | `summary` | `String` | no | 2–4 sentences. |
| | `completedAt` | `Instant` | no | When `CodingAgent` returned. |
| `MonitorAlert` | `alertId` | `String` | no | UUID minted by `RuntimeMonitor`. |
| | `kind` | `AlertKind` | no | Enum value. |
| | `detail` | `String` | no | Human-readable description of the anomaly. |
| | `alertedAt` | `Instant` | no | When `RuntimeMonitor` detected the pattern. |
| `AgentTask` (entity state) | `taskId` | `String` | no | — |
| | `issueRef` | `Optional<IssueRef>` | yes | Populated after `TaskSubmitted`. |
| | `issueBody` | `Optional<IssueBody>` | yes | Populated after `IssueFetched`. |
| | `plan` | `Optional<AgentPlan>` | yes | Populated after `PlanProposed`. |
| | `outcome` | `Optional<TaskOutcome>` | yes | Populated after `OutcomeRecorded`. |
| | `toolTrace` | `List<ToolCall>` | no | Grows with each `ToolCallExecuted` event. |
| | `alerts` | `List<MonitorAlert>` | no | Grows with each `MonitorAlertRaised` event. |
| | `status` | `AgentTaskStatus` | no | See enum. |
| | `haltRequested` | `boolean` | no | Set to `true` by `HaltRequested` event. |
| | `createdAt` | `Instant` | no | When `TaskSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `AgentTask` is `Optional<T>`. The `toolTrace` and `alerts` lists are never `Optional` — they start empty and grow. The view's table updater wraps `Optional` fields with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ToolCallStatus`: `ALLOWED`, `BLOCKED`, `SUCCEEDED`, `FAILED`.
`OutcomeStatus`: `SUCCEEDED`, `PARTIAL`, `FAILED`.
`AlertKind`: `RATE_SPIKE`, `REPEATED_FAILURE`, `SHELL_PATTERN`.
`AgentTaskStatus`: `SUBMITTED`, `ISSUE_FETCHED`, `PLAN_READY`, `CHECKPOINT_PENDING`, `EXECUTING`, `PATCH_READY`, `HALTED`, `FAILED`.

## Events (`AgentTaskEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskSubmitted` | `issueRef` | → SUBMITTED |
| `IssueFetched` | `issueBody` | → ISSUE_FETCHED |
| `PlanProposed` | `plan` | → PLAN_READY |
| `CheckpointPending` | — | → CHECKPOINT_PENDING |
| `CheckpointApproved` | — | → EXECUTING |
| `CheckpointRejected` | — | → FAILED |
| `ExecutionStarted` | — | (internal; no status change) |
| `ToolCallExecuted` | `toolCall` | (no status change; appended to `toolTrace`) |
| `OutcomeRecorded` | `outcome` | → PATCH_READY |
| `HaltRequested` | — | (sets `haltRequested = true`; no status change) |
| `TaskHalted` | — | → HALTED (terminal) |
| `TaskFailed` | `reason: String` | → FAILED (terminal) |
| `MonitorAlertRaised` | `alert` | (no status change; appended to `alerts`) |

`emptyState()` returns `AgentTask.initial("")` with all `Optional` fields as `Optional.empty()`, `toolTrace = List.of()`, `alerts = List.of()`, `haltRequested = false`, and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AgentTaskRow` mirrors `AgentTask`. The view declares ONE query: `getAllTasks: SELECT * AS tasks FROM agent_task_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`AgentTasks.java`)

```java
public final class AgentTasks {
  public static final Task<AgentPlan> PLAN_ISSUE = Task
      .name("Plan issue fix")
      .description("Read the attached GitHub issue and produce an AgentPlan — file list and approach, no writes.")
      .resultConformsTo(AgentPlan.class);

  public static final Task<TaskOutcome> RESOLVE_ISSUE = Task
      .name("Resolve issue")
      .description("Implement the approved plan using readFile, writeFile, runShell, searchCodebase tools and return a TaskOutcome.")
      .resultConformsTo(TaskOutcome.class);

  private AgentTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7). Both constants are required: `PLAN_ISSUE` for the plan phase invocation in `AgentTaskWorkflow.planStep` and `RESOLVE_ISSUE` for the execute phase invocation in `AgentTaskWorkflow.executeStep`.
