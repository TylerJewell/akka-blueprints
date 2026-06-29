# Data model — context-preset-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PresetDefinition` | `presetId` | `String` | no | `"<env>:<role>"` — the registry key. |
| | `environment` | `String` | no | `"dev"` / `"staging"` / `"prod"`. |
| | `role` | `String` | no | `"admin"` / `"guest"`. |
| | `modelId` | `String` | no | Model id to use for this preset (e.g. `"claude-sonnet-4-6"`). |
| | `allowedTools` | `List<String>` | no | Tool names this preset may invoke. |
| | `instructionAddendum` | `String` | no | Additional system-prompt text prepended for this preset. |
| `PresetRequest` | `requestId` | `String` | no | UUID minted by `PresetRequestEndpoint`. |
| | `environment` | `String` | no | Caller-supplied environment. |
| | `role` | `String` | no | Caller-supplied role. |
| | `requestText` | `String` | no | The freetext request body. |
| | `submittedBy` | `String` | no | Caller identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCallLogEntry` | `toolName` | `String` | no | Name of the tool attempted. |
| | `status` | `ToolCallStatus` | no | `INVOKED` or `BLOCKED`. |
| | `inputSummary` | `String` | no | Brief description of the tool call arguments. |
| | `outputSummary` | `String` | no | Tool result summary or block reason. |
| | `calledAt` | `Instant` | no | When the tool call was attempted. |
| `PresetRequestResult` | `answerText` | `String` | no | Agent's answer to the caller's request. |
| | `toolCallLog` | `List<ToolCallLogEntry>` | no | Full log of all attempted tool calls. |
| | `completedAt` | `Instant` | no | When the agent returned. |
| `AuditEntry` | `requestId` | `String` | no | — |
| | `presetId` | `String` | no | Resolved preset id at execution time. |
| | `toolCallCount` | `int` | no | Total tool call attempts (INVOKED + BLOCKED). |
| | `blockedToolCallCount` | `int` | no | Count of BLOCKED entries in `toolCallLog`. |
| | `auditedAt` | `Instant` | no | When `auditStep` completed. |
| `PresetRequestState` (entity state) | `requestId` | `String` | no | — |
| | `request` | `Optional<PresetRequest>` | yes | Populated after `RequestSubmitted`. |
| | `resolvedPreset` | `Optional<PresetDefinition>` | yes | Populated after `PresetResolved`. |
| | `result` | `Optional<PresetRequestResult>` | yes | Populated after `RequestCompleted`. |
| | `audit` | `Optional<AuditEntry>` | yes | Populated after `RequestAudited`. |
| | `status` | `RequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RequestSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `PresetRequestState` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ToolCallStatus`: `INVOKED`, `BLOCKED`.
`RequestStatus`: `SUBMITTED`, `PRESET_RESOLVED`, `EXECUTING`, `COMPLETED`, `FAILED`.

## Events (`PresetRequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RequestSubmitted` | `request` | → SUBMITTED |
| `PresetResolved` | `preset` | → PRESET_RESOLVED |
| `ExecutionStarted` | — | → EXECUTING |
| `RequestCompleted` | `result` | → COMPLETED |
| `RequestAudited` | `audit` | COMPLETED (additive) |
| `RequestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `PresetRequestState.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PresetRequestRow` mirrors `PresetRequestState` in full — there are no fields stripped for the view (unlike the docreview pattern, there is no raw payload equivalent to `rawDocument` that must be withheld from the read model). The view declares ONE query: `getAllRequests: SELECT * AS requests FROM preset_request_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`PresetRequestTasks.java`)

```java
public final class PresetRequestTasks {
  public static final Task<PresetRequestResult> EXECUTE_PRESET_REQUEST = Task
      .name("Execute preset request")
      .description("Resolve the caller preset, execute the request using allowed tools, and return a PresetRequestResult")
      .resultConformsTo(PresetRequestResult.class);

  private PresetRequestTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## PresetRegistry (Key-Value Entity)

State type: `PresetDefinition`. Key: `"<env>:<role>"` (e.g. `"prod:admin"`). Six seeded entries at startup. Commands: `upsertPreset`, `getPreset`, `deletePreset`. The entity is read-only from the workflow's perspective — the request lifecycle never modifies it.
