# Data model — hook-instrumentation

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ToolSpec` | `toolName` | `String` | no | Unique tool identifier used in hook log entries. |
| | `description` | `String` | no | Human-readable description of what the tool does. |
| | `allowedCallers` | `List<String>` | no | Persona IDs permitted to invoke this tool. |
| `TaskRequest` | `observationId` | `String` | no | UUID minted by `ObservationEndpoint`. |
| | `taskDescription` | `String` | no | User-supplied task to execute. |
| | `personaId` | `String` | no | Active persona (e.g. `observer-standard`). |
| | `availableTools` | `List<ToolSpec>` | no | Tools the agent may call. |
| | `simulateBlockedTool` | `boolean` | no | When true, the seeded blocked tool is included in the attempt path. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `HookConfig` | `blockedToolNames` | `List<String>` | no | Tool names that `BeforeToolCallGuardrail` will reject. |
| | `sensitivePatterns` | `List<String>` | no | Regex patterns for `AfterToolCallGuardrail` redaction. |
| | `credentialPatterns` | `List<String>` | no | Regex patterns for `BeforeLlmCallGuardrail` stripping. |
| `HookLogEntry` | `entryId` | `String` | no | UUID for this entry. |
| | `hookPoint` | `HookPoint` | no | Enum value. |
| | `toolNameOrPhase` | `String` | no | Tool name or `"llm-invocation-N"`. |
| | `verdict` | `HookVerdict` | no | Enum value. |
| | `originalPayloadHash` | `String` | no | SHA-256 hex when verdict is `REDACTED`; empty string otherwise. |
| | `sanitizedPayload` | `String` | no | What the agent actually saw after the hook ran. |
| | `rejectReason` | `String` | no | Non-empty when verdict is `BLOCKED`; empty string otherwise. |
| | `firedAt` | `Instant` | no | When the hook fired. |
| `AgentOutcome` | `outcome` | `Outcome` | no | Enum value. |
| | `responseText` | `String` | no | 1–5 sentence summary of the task result. |
| | `hookLog` | `List<HookLogEntry>` | no | Full hook log across all iterations. |
| | `completedAt` | `Instant` | no | When the agent returned. |
| `CoverageScore` | `score` | `int` | no | 1–5. |
| | `note` | `String` | no | One sentence describing coverage completeness. |
| | `scoredAt` | `Instant` | no | When `CoverageScorer` finished. |
| `Observation` (entity state) | `observationId` | `String` | no | — |
| | `request` | `Optional<TaskRequest>` | yes | Populated after `TaskSubmitted`. |
| | `hookConfig` | `Optional<HookConfig>` | yes | Populated after `HookChainInitialized`. |
| | `outcome` | `Optional<AgentOutcome>` | yes | Populated after `OutcomeRecorded`. |
| | `coverage` | `Optional<CoverageScore>` | yes | Populated after `CoverageScored`. |
| | `status` | `ObservationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TaskSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Observation` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`HookPoint`: `BEFORE_TOOL_CALL`, `AFTER_TOOL_CALL`, `BEFORE_LLM_CALL`.
`HookVerdict`: `ALLOWED`, `BLOCKED`, `REDACTED`, `PASS_THROUGH`.
`Outcome`: `SUCCESS`, `PARTIAL`, `BLOCKED`.
`ObservationStatus`: `SUBMITTED`, `HOOK_CHAIN_READY`, `EXECUTING`, `COMPLETED`, `FAILED`.

## Events (`ObservationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskSubmitted` | `request` | → SUBMITTED |
| `HookChainInitialized` | `hookConfig` | → HOOK_CHAIN_READY |
| `ExecutionStarted` | — | → EXECUTING |
| `OutcomeRecorded` | `outcome` | → (no transition; coverage step follows) |
| `CoverageScored` | `coverage` | → COMPLETED (terminal happy) |
| `ObservationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Observation.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ObservationRow` mirrors `Observation` minus `request.taskDescription` full text and `hookConfig` raw patterns (summarized for the list view). Full detail on demand via `GET /api/observations/{id}`.

The view declares ONE query: `getAllObservations: SELECT * AS observations FROM observation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ObservationTasks.java`)

```java
public final class ObservationTasks {
  public static final Task<AgentOutcome> EXECUTE_TASK = Task
      .name("Execute task")
      .description("Execute the described task using the available tools; log every hook firing into the hookLog; return an AgentOutcome")
      .resultConformsTo(AgentOutcome.class);

  private ObservationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Coverage scoring rubric (`CoverageScorer.java`)

| Condition | Points lost |
|---|---|
| Any tool call lacks a matching `BEFORE_TOOL_CALL` entry | −1 per missing entry (max −2) |
| Any tool call lacks a matching `AFTER_TOOL_CALL` entry | −1 per missing entry (max −2) |
| No `BEFORE_LLM_CALL` entry exists despite at least one LLM invocation | −1 |
| A `BLOCKED` entry has an empty `rejectReason` | −1 |

Scoring starts at 5 and subtracts. Minimum score is 1. A fully instrumented observation with all pairs present and all reasons populated scores 5.
