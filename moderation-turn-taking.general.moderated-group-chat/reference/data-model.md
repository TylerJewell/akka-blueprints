# Data model

Every Java record, event, and enum the generated system defines. Lifecycle fields that are null until a later event fires are `Optional<T>` (Lesson 6); the turn history is a `List` that is empty, never null.

## `ChatSession` — entity state and View row

`domain/ChatSession.java`. This single record is both the `ChatSessionEntity` state and the `SessionsView` row type.

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Session id; equals the workflow id. |
| `topic` | `Optional<String>` | until started | The subject of the group chat session. |
| `status` | `ChatSessionStatus` | no | Lifecycle stage. |
| `currentTurn` | `int` | no | Current turn number, 0 before start, 1–20 while chatting. |
| `turns` | `List<TurnLine>` | no (empty) | Append-only history of every assistant turn. |
| `terminationReason` | `Optional<String>` | until concluded | `"CONSENSUS"`, `"MAX_TURNS_REACHED"`, or `"ESCALATED"`. |
| `conclusionSummary` | `Optional<String>` | until concluded | Synthesis of agreed points; non-empty on `CONSENSUS`. |
| `startedAt` | `Optional<Instant>` | until started | When the session began. |
| `concludedAt` | `Optional<Instant>` | until concluded | When the Orchestrator concluded. |
| `escalatedAt` | `Optional<Instant>` | until escalated | When the stall monitor or guardrail escalated it. |
| `qualityScore` | `Optional<Double>` | until evaluated | Score from `SessionQualityEvaluator`, 0.0–1.0. |
| `qualityNotes` | `Optional<String>` | until evaluated | One-line explanation of the score. |

Helpers: `static ChatSession initial(String id)` returns `status = CREATED`, `currentTurn = 0`, `turns = List.of()`, every `Optional` empty — and references no `commandContext()` (Lesson 3). `ChatSession applyEvent(SessionEvent e)` is a per-variant switch returning the next state.

## `TurnLine` — one assistant message in the history

`domain/TurnLine.java`.

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `turn` | `int` | no | The turn number this message belongs to. |
| `assistant` | `String` | no | `"RESEARCHER"` or `"CRITIC"`. |
| `message` | `String` | no | The assistant's message text. |
| `flagged` | `boolean` | no | Whether the guardrail flagged this turn as problematic. |
| `at` | `Instant` | no | When the turn was recorded. |

## `ChatSessionStatus` — enum

```
CREATED · CHATTING · CONCLUDED · ESCALATED
```

The termination-reason distinction lives in the `terminationReason` string, not the enum, so no view query indexes the enum column (Lesson 2). `CONCLUDED` is reached on `CONSENSUS`, `MAX_TURNS_REACHED`, or a guardrail-triggered escalation that goes through `concludeStep`.

## `SessionEvent` — sealed interface, six variants

`domain/SessionEvent.java`. Each variant carries `Instant timestamp()`.

| Event | Trigger | Carries |
|---|---|---|
| `SessionStarted` | `openStep` writes the opening state | `id, topic, timestamp` |
| `TurnRecorded` | a researcher or critic turn completes | `id, turn, assistant, message, flagged, timestamp` |
| `RoundAdvanced` | Orchestrator returns `CONTINUE` | `id, newTurn, timestamp` |
| `SessionConcluded` | `concludeStep` runs | `id, terminationReason, summary (nullable), turnsUsed, timestamp` |
| `SessionEscalated` | `StalledSessionMonitor` fires | `id, timestamp` |
| `QualityEvaluated` | `SessionQualityEvaluator` scores a concluded session | `id, score, notes, timestamp` |

```java
sealed interface SessionEvent
    permits SessionStarted, TurnRecorded, RoundAdvanced,
            SessionConcluded, SessionEscalated, QualityEvaluated {
  Instant timestamp();
}
```

`SessionConcluded.summary` is nullable in the event (no synthesis on a `MAX_TURNS_REACHED` or `ESCALATED` conclusion unless the Orchestrator provided one); the entity wraps it into the `Optional` row field when applying.

## Agent result records

`application/ChatTurn.java` and `application/OrchestratorDecision.java` — the typed results the agents return, retrieved via `forTask(taskId).result(...)`.

```java
record ChatTurn(String message,     // assistant's contribution; non-empty
                String rationale,   // one sentence explaining the turn
                boolean flagged)    // true only for harmful or off-topic content
{}

record OrchestratorDecision(String verdict,    // "CONTINUE" | "CONCLUDE" | "ESCALATE"
                             String summary,   // non-empty when verdict is CONCLUDE
                             String reasoning) // one sentence explaining the verdict
{}
```

## Request and queue records

```java
record StartRequest(String topic) {}   // POST /api/sessions body
record SessionRequestQueued(String topic, Instant at) {}  // SessionRequestQueue event
```

## `ChatTasks` — companion to the AutonomousAgents (Lesson 7)

`application/ChatTasks.java` declares the three `Task<R>` constants the agents accept. Generating an AutonomousAgent without this file is a compile error.

```java
public final class ChatTasks {
  public static final Task<ChatTurn> RESEARCHER_TURN = Task
    .name("Researcher turn").description("Produce the researcher's contribution for the current turn")
    .resultConformsTo(ChatTurn.class);
  public static final Task<ChatTurn> CRITIC_TURN = Task
    .name("Critic turn").description("Produce the critic's evaluation for the current turn")
    .resultConformsTo(ChatTurn.class);
  public static final Task<OrchestratorDecision> ORCHESTRATE = Task
    .name("Orchestrate round").description("Decide continue, conclude, or escalate after both assistants moved")
    .resultConformsTo(OrchestratorDecision.class);
}
```

## `SystemControl` state

`application/SystemControl.java` — a `KeyValueEntity`, single instance `"default"`, holding `record HaltState(boolean halted) {}`. Commands `halt`, `resume`, `isHalted`.

## View row note

`SessionsView` projects `ChatSessionEntity` events into a row of type `ChatSession` (the same record above). The one query is `getAllSessions` → `SELECT * AS sessions FROM sessions_view`; a `streamAllSessions` variant feeds the SSE endpoint. No `WHERE status` clause — callers filter by status client-side (Lesson 2). Because the row type carries `Optional<T>` lifecycle fields, the view materializer accepts it (Lesson 6).
