# Data model — persona-hot-reload

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PersonaPayload` | `agentRole` | `String` | no | The identity the agent holds in this deployment. |
| | `agentGoal` | `String` | no | The primary objective of the agent's responses. |
| | `agentInstructions` | `String` | no | Behavioral rules and constraints for this persona. |
| | `agentModel` | `Optional<String>` | yes | Model override; absent means "keep current default". |
| `PersonaSnapshot` | `changeId` | `String` | no | UUID minted by `PersonaEndpoint`. |
| | `payload` | `PersonaPayload` | no | The validated persona payload. |
| | `pushedBy` | `String` | no | Operator identifier. |
| | `pushedAt` | `Instant` | no | When the endpoint received the push. |
| | `rejectionReason` | `Optional<String>` | yes | Populated only on rejected changes. |
| `ProbeResult` | `probeId` | `String` | no | Stable id from `probe-set.jsonl`. |
| | `question` | `String` | no | The probe question sent to the agent. |
| | `response` | `String` | no | The agent's response text. |
| | `outcome` | `ProbeOutcome` | no | Enum value. |
| | `rationale` | `String` | no | One-sentence scorer explanation. |
| `RevalidationResult` | `status` | `RevalidationStatus` | no | Enum value. |
| | `probeResults` | `List<ProbeResult>` | no | One entry per probe in the set. |
| | `evaluatedAt` | `Instant` | no | When `BehavioralRevalidator` finished. |
| `MonitoringWindow` | `forwardedCount` | `int` | no | Responses forwarded to operator channel. |
| | `open` | `boolean` | no | True while the 300-second window is active. |
| | `openedAt` | `Instant` | no | When `MonitoringWindowOpened` was emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Populated when window closes. |
| `AgentResponse` | `changeId` | `String` | no | Active persona's changeId at time of response. |
| | `queryId` | `String` | no | Caller-supplied query identifier. |
| | `responseText` | `String` | no | The agent's response text. |
| | `respondedAt` | `Instant` | no | When the agent returned. |
| `ValidationResult` | `valid` | `boolean` | no | Gate outcome. |
| | `rejectionReason` | `Optional<String>` | yes | First failing check description if `valid=false`. |
| `PersonaChange` (entity state) | `changeId` | `String` | no | — |
| | `snapshot` | `Optional<PersonaSnapshot>` | yes | Populated after `PersonaChangeProposed`. |
| | `revalidation` | `Optional<RevalidationResult>` | yes | Populated after `PersonaRevalidated`. |
| | `monitoring` | `Optional<MonitoringWindow>` | yes | Populated after `MonitoringWindowOpened`. |
| | `status` | `PersonaChangeStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PersonaChangeProposed` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `PersonaChange` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ProbeOutcome`: `PASS`, `FAIL`, `INCONCLUSIVE`.
`RevalidationStatus`: `PASS`, `PARTIAL`, `FAIL`.
`PersonaChangeStatus`: `VALIDATING`, `REJECTED`, `ACTIVATING`, `ACTIVE`, `REVALIDATING`, `MONITORED`, `FAILED`.

## Events (`PersonaEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PersonaChangeProposed` | `snapshot` | → VALIDATING |
| `PersonaActivated` | `snapshot` | → ACTIVATING |
| `PersonaRejected` | `reason: String` | → REJECTED (terminal) |
| `PersonaRevalidated` | `result` | → REVALIDATING → MONITORED (after window closes) |
| `MonitoringWindowOpened` | `openedAt: Instant` | (ACTIVE → window open; status stays REVALIDATING until PersonaRevalidated) |
| `MonitoringWindowClosed` | `forwardedCount: int` | → MONITORED (terminal happy) |
| `PersonaChangeFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `PersonaChange.initial("")` with all `Optional` fields as `Optional.empty()` and `status = VALIDATING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PersonaChangeRow` mirrors `PersonaChange`. The view declares ONE query: `getAllChanges: SELECT * AS changes FROM persona_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side. `GET /api/personas/current` is implemented by filtering `getAllChanges` for the most recent row with `status == MONITORED` or `status == ACTIVE`.

## Task definition (`PersonaTasks.java`)

```java
public final class PersonaTasks {
  public static final Task<AgentResponse> ANSWER_QUERY = Task
      .name("Answer query")
      .description("Answer the user message using the active persona definition")
      .resultConformsTo(AgentResponse.class);

  private PersonaTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Persona fixtures (seed data)

Three seeded personas in `src/main/resources/sample-events/persona-fixtures.jsonl`:

| Fixture | agentRole | agentGoal | Notes |
|---|---|---|---|
| `assistant` | General-purpose support assistant | Resolve queries accurately and completely | Broad instructions, no hard scope restrictions. |
| `escalation` | Issue escalation specialist | Identify unresolved issues and route to human agents | Narrow: escalate; do not resolve independently. |
| `restricted` | Read-only policy reference assistant | Summarize existing policy documents only | No advice, no speculation, no recommendations. Designed to produce a FAIL on advisory probes. |
