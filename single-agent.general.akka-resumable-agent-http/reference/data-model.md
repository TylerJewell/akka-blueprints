# Data model — akka-resumable-agent-http

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `RunRequest` | `runId` | `String` | no | UUID minted by `AgentEndpoint`. |
| | `location` | `String` | no | User-supplied location string. |
| | `queryType` | `QueryType` | no | Enum value. |
| | `requestedBy` | `String` | no | User identifier. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCallAttempt` | `toolName` | `String` | no | Always `"SlowWeatherTool"`. |
| | `argumentLocation` | `String` | no | The location passed to the tool (may differ from `request.location` after a guardrail-corrected retry). |
| | `queryType` | `QueryType` | no | Enum value passed to the tool. |
| | `calledAt` | `Instant` | no | When the tool was invoked. |
| | `guardrailRejected` | `boolean` | no | Whether any iteration was blocked by `ToolCallGuardrail`. |
| | `rejectionReason` | `String` | yes | Check name that failed; null when `guardrailRejected == false`. |
| `WeatherReport` | `location` | `String` | no | Echoed from input. |
| | `queryType` | `QueryType` | no | Echoed from input. |
| | `summary` | `String` | no | 1–2-sentence description. |
| | `temperatureCelsius` | `double` | no | Current or forecast-high temperature. |
| | `conditions` | `String` | no | e.g. `"Sunny"`, `"Rain"`, `"Fog then Sunny"`. |
| | `reportedAt` | `Instant` | no | When the tool returned. |
| `IncidentEvent` | `runId` | `String` | no | The affected run. |
| | `interruptedStep` | `String` | no | Step name: `initStep`, `toolCallStep`, or `reportStep`. |
| | `elapsedBeforeCrashMs` | `long` | no | Milliseconds from run creation to crash detection. |
| | `severity` | `IncidentSeverity` | no | Enum value. |
| | `notes` | `String` | no | Human-readable description. |
| | `detectedAt` | `Instant` | no | When `IncidentEvaluator` ran. |
| `AgentRun` (entity state) | `runId` | `String` | no | — |
| | `request` | `Optional<RunRequest>` | yes | Populated after `RunInitiated`. |
| | `toolCall` | `Optional<ToolCallAttempt>` | yes | Populated after `ToolCallStarted`. |
| | `report` | `Optional<WeatherReport>` | yes | Populated after `ToolCallCompleted`. |
| | `incident` | `Optional<IncidentEvent>` | yes | Populated after `RunResumed` + `incidentStep`. |
| | `status` | `AgentRunStatus` | no | See enum. |
| | `resumeCount` | `int` | no | 0 on first run; incremented by each `RunResumed`. |
| | `createdAt` | `Instant` | no | When `RunInitiated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `AgentRun` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryType`: `CURRENT`, `FORECAST`, `AIR_QUALITY`.
`AgentRunStatus`: `INITIATED`, `TOOL_CALLED`, `REPORTING`, `RESUMED`, `COMPLETED`, `FAILED`.
`IncidentSeverity`: `INFO`, `WARNING`, `ERROR`.

## Events (`AgentRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunInitiated` | `request` | → INITIATED |
| `ToolCallStarted` | `location, queryType` | → TOOL_CALLED |
| `ToolCallCompleted` | `report` (WeatherReport) | → REPORTING |
| `ReportGenerated` | `report` (final) | stays REPORTING |
| `RunCompleted` | — | → COMPLETED (terminal happy) |
| `RunFailed` | `reason: String` | → FAILED (terminal) |
| `RunResumed` | `resumeCount: int` | → RESUMED (transient; workflow re-enters from last step) |

`emptyState()` returns `AgentRun.initial("")` with all `Optional` fields as `Optional.empty()`, `resumeCount = 0`, and `status = INITIATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AgentRunRow` mirrors `AgentRun` minus the full `RunRequest` fields — keeps `runId`, `location` (from request), `queryType`, `status`, `resumeCount`, `report`, `incident`, `createdAt`, `finishedAt`, `guardrailRejected`, `rejectionReason`.

The view declares ONE query: `getAllRuns: SELECT * AS runs FROM agent_run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AgentTasks.java`)

```java
public final class AgentTasks {
  public static final Task<WeatherReport> QUERY_WEATHER = Task
      .name("Query weather")
      .description("Call SlowWeatherTool with the supplied location and return a WeatherReport")
      .resultConformsTo(WeatherReport.class);

  private AgentTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
