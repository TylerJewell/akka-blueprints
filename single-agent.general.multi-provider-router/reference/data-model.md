# Data model — multi-provider-router

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CallRequest` | `callId` | `String` | no | UUID minted by `RouterEndpoint`. |
| | `promptText` | `String` | no | The user's raw prompt. |
| | `hint` | `ProviderHint` | no | `AUTO`, `OPENAI`, or `ANTHROPIC`. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `CallResult` | `selectedProvider` | `Provider` | no | Which backend handled the call. |
| | `responseText` | `String` | no | The agent's answer. |
| | `latencyMs` | `long` | no | Round-trip milliseconds, measured by the workflow. |
| | `completedAt` | `Instant` | no | When the agent returned. |
| `CallRecord` | `callId` | `String` | no | — |
| (entity state) | `request` | `Optional<CallRequest>` | yes | Populated after `CallSubmitted`. |
| | `result` | `Optional<CallResult>` | yes | Populated after `CallCompleted`. |
| | `status` | `CallStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CallSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `errorReason` | `Optional<String>` | yes | Populated after `CallFailed`. |
| `ProviderStats` | `provider` | `Provider` | no | The provider being described. |
| | `totalCalls` | `int` | no | Calls in the window (includes failed). |
| | `failedCalls` | `int` | no | Failed calls in the window. |
| | `p50LatencyMs` | `long` | no | 50th-percentile latency. |
| | `p95LatencyMs` | `long` | no | 95th-percentile latency. |
| | `avgQualityScore` | `double` | no | Rule-based score 1.0–5.0. |
| `EvalReport` | `windowId` | `String` | no | e.g. `"window-3"`. |
| | `windowSize` | `int` | no | Number of calls in this window. |
| | `stats` | `List<ProviderStats>` | no | One entry per provider. |
| | `generatedAt` | `Instant` | no | When `EvaluationAggregator` finished. |
| `MonitorRow` | `windowId` | `String` | no | — |
| | `windowSize` | `int` | no | — |
| | `stats` | `List<ProviderStats>` | no | — |
| | `generatedAt` | `Instant` | no | — |

Every nullable field on `CallRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Provider`: `OPENAI`, `ANTHROPIC`, `MOCK`.
`ProviderHint`: `AUTO`, `OPENAI`, `ANTHROPIC`.
`CallStatus`: `PENDING`, `DISPATCHED`, `COMPLETED`, `FAILED`.

## Events (`CallEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CallSubmitted` | `request` | → PENDING |
| `CallDispatched` | — | → DISPATCHED |
| `CallCompleted` | `result` | → COMPLETED (terminal happy) |
| `CallFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `CallRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = PENDING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View rows

`CallView` row type `CallRow` mirrors `CallRecord` exactly. The view declares ONE query: `getAllCalls: SELECT * AS calls FROM call_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

`MonitorView` row type `MonitorRow` mirrors `EvalReport`. The view receives upsert calls from `PerformanceMonitor` (not from entity events). ONE query: `getAllReports: SELECT * AS reports FROM monitor_view`.

## Task definition (`RouterTasks.java`)

```java
public final class RouterTasks {
  public static final Task<CallResult> ROUTE_PROMPT = Task
      .name("Route prompt")
      .description("Dispatch the prompt to the configured provider and return a CallResult")
      .resultConformsTo(CallResult.class);

  private RouterTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
