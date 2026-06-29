# Data model

Every Java record, event, and enum the generated system defines. Lifecycle fields that are null until a later event fires are `Optional<T>` (Lesson 6); the turn history is a `List` that is empty, never null.

## `Collaboration` — entity state and View row

`domain/Collaboration.java`. This single record is both the `CollaborationEntity` state and the `CollaborationsView` row type.

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Collaboration id; equals the workflow id. |
| `taskDescription` | `Optional<String>` | until started | The task to be solved. |
| `assistantRole` | `Optional<String>` | until started | Role label assigned to the AI Assistant. |
| `userRole` | `Optional<String>` | until started | Role label assigned to the AI User. |
| `status` | `CollaborationStatus` | no | Lifecycle stage. |
| `currentRound` | `int` | no | Current round, 0 before start, 1–12 while collaborating. |
| `turns` | `List<TurnLine>` | no (empty) | Append-only history of every dialogue turn. |
| `latestAssistantMessage` | `Optional<String>` | until first assistant turn | Most recent assistant message. |
| `latestUserMessage` | `Optional<String>` | until first user turn | Most recent user message. |
| `outcome` | `Optional<String>` | until concluded | `"SOLVED"` or `"IMPASSE"`. |
| `solutionSummary` | `Optional<String>` | until solved | One-paragraph summary of what was solved; set only on SOLVED. |
| `finalAnswer` | `Optional<String>` | until solved | Complete answer extracted from the assistant's turns; set only on SOLVED. |
| `startedAt` | `Optional<Instant>` | until started | When the collaboration began. |
| `concludedAt` | `Optional<Instant>` | until concluded | When the Coordinator concluded. |
| `escalatedAt` | `Optional<Instant>` | until escalated | When the stall monitor escalated it. |
| `outcomeScore` | `Optional<Double>` | until evaluated | Score from `SolutionEvaluator`, 0.0–1.0. |
| `outcomeNotes` | `Optional<String>` | until evaluated | One-line explanation of the score. |

Helpers: `static Collaboration initial(String id)` returns `status = CREATED`, `currentRound = 0`, `turns = List.of()`, every `Optional` empty — and references no `commandContext()` (Lesson 3). `Collaboration applyEvent(CollaborationEvent e)` is a per-variant switch returning the next state.

## `TurnLine` — one dialogue turn in the history

`domain/TurnLine.java`.

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `round` | `int` | no | The round this turn belongs to. |
| `agent` | `String` | no | `"ASSISTANT"` or `"USER"`. |
| `message` | `String` | no | The full message content of this turn. |
| `intent` | `String` | no | One-phrase description of the turn's intent. |
| `at` | `Instant` | no | When the turn was recorded. |

## `CollaborationStatus` — enum

```
CREATED · COLLABORATING · CONCLUDED · ESCALATED
```

The solved-versus-impasse split lives in the `outcome` string, not the enum, so no view query indexes the enum column (Lesson 2). `CONCLUDED` is reached on either a solution or an impasse.

## `CollaborationEvent` — sealed interface, six variants

`domain/CollaborationEvent.java`. Each variant carries `Instant timestamp()`.

| Event | Trigger | Carries |
|---|---|---|
| `CollaborationStarted` | `openStep` writes the opening state | `id, taskDescription, assistantRole, userRole, timestamp` |
| `TurnRecorded` | a user or assistant turn completes | `id, round, agent, message, intent, timestamp` |
| `RoundAdvanced` | Coordinator returns `CONTINUE` | `id, newRound, timestamp` |
| `CollaborationConcluded` | `concludeStep` runs | `id, outcome, solutionSummary (nullable), finalAnswer (nullable), roundsUsed, timestamp` |
| `CollaborationEscalated` | `StalledCollaborationMonitor` fires | `id, timestamp` |
| `OutcomeEvaluated` | `SolutionEvaluator` scores a concluded collaboration | `id, score, notes, timestamp` |

```java
sealed interface CollaborationEvent
    permits CollaborationStarted, TurnRecorded, RoundAdvanced,
            CollaborationConcluded, CollaborationEscalated, OutcomeEvaluated {
  Instant timestamp();
}
```

`CollaborationConcluded.solutionSummary` and `.finalAnswer` are nullable in the event (no solution on an impasse); the entity wraps them into the `Optional` row fields when applying.

## Agent result records

`application/DialogueTurn.java` and `application/CoordinatorDecision.java` — the typed results the agents return, retrieved via `forTask(taskId).result(...)`.

```java
record DialogueTurn(String message,     // the agent's response this turn
                    String intent,      // one-phrase intent label
                    boolean taskComplete) {}

record CoordinatorDecision(String verdict,         // "CONTINUE" | "SOLVED" | "IMPASSE"
                           String solutionSummary, // "" unless verdict is SOLVED
                           String finalAnswer,      // "" unless verdict is SOLVED
                           String reasoning) {}
```

## Request and queue records

```java
record StartRequest(String taskDescription, String assistantRole, String userRole) {}  // POST /api/collaborations body
record TaskQueued(String taskDescription, String assistantRole, String userRole, Instant at) {}  // InboundTaskQueue event
```

## `CollaborationTasks` — companion to the AutonomousAgents (Lesson 7)

`application/CollaborationTasks.java` declares the three `Task<R>` constants the agents accept. Generating an AutonomousAgent without this file is a compile error.

```java
public final class CollaborationTasks {
  public static final Task<DialogueTurn> ASSISTANT_TURN = Task
    .name("Assistant turn").description("Produce the assistant's dialogue turn for the current round")
    .resultConformsTo(DialogueTurn.class);
  public static final Task<DialogueTurn> USER_TURN = Task
    .name("User turn").description("Produce the user agent's instruction or question for the current round")
    .resultConformsTo(DialogueTurn.class);
  public static final Task<CoordinatorDecision> COORDINATE = Task
    .name("Coordinate round").description("Decide continue, solved, or impasse after both parties moved")
    .resultConformsTo(CoordinatorDecision.class);
}
```

## `SystemControl` state

`application/SystemControl.java` — a `KeyValueEntity`, single instance `"default"`, holding `record HaltState(boolean halted) {}`. Commands `halt`, `resume`, `isHalted`.

## View row note

`CollaborationsView` projects `CollaborationEntity` events into a row of type `Collaboration` (the same record above). The one query is `getAllCollaborations` → `SELECT * AS collaborations FROM collaborations_view`; a `streamAllCollaborations` variant feeds the SSE endpoint. No `WHERE status` clause — callers filter by status client-side (Lesson 2). Because the row type carries `Optional<T>` lifecycle fields, the view materializer accepts it (Lesson 6).
