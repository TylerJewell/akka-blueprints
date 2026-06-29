# Data model — inline-agent-demo

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `AgentDefinition` | `agentName` | `String` | no | Caller-supplied label for this agent variant. |
| | `systemPrompt` | `String` | no | Instructions prepended (after base prompt) for this run. |
| | `outputSchema` | `String` | no | JSON Schema string describing the expected answer shape. |
| | `allowedTools` | `List<String>` | no | Tool names the agent may invoke. Empty list = no tools. |
| `InlineAgentRequest` | `runId` | `String` | no | UUID minted by `RunEndpoint`. |
| | `question` | `String` | no | The caller's question or task text. |
| | `agentDefinition` | `AgentDefinition` | no | The inline definition for this run. |
| | `submittedBy` | `String` | no | Caller identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `AgentResponse` | `answer` | `String` | no | Structured text or JSON matching `outputSchema`. |
| | `tokenCount` | `int` | no | Approximate token count of the response. |
| | `answeredAt` | `Instant` | no | When `InlineAgentRunner` returned. |
| `Run` (entity state) | `runId` | `String` | no | — |
| | `request` | `Optional<InlineAgentRequest>` | yes | Populated after `RunReceived`. |
| | `response` | `Optional<AgentResponse>` | yes | Populated after `RunCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated after `RunFailed`. |
| | `status` | `RunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Run` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`RunStatus`: `RECEIVED`, `RUNNING`, `COMPLETED`, `FAILED`.

## Events (`RunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunReceived` | `request` | → RECEIVED |
| `RunStarted` | — | → RUNNING |
| `RunCompleted` | `response` | → COMPLETED (terminal happy) |
| `RunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Run.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RunRow` mirrors `Run`. The view declares ONE query: `getAllRuns: SELECT * AS runs FROM run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`RunTasks.java`)

```java
public final class RunTasks {
  public static final Task<AgentResponse> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Read the question and the agent definition and produce an AgentResponse")
      .resultConformsTo(AgentResponse.class);

  private RunTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
