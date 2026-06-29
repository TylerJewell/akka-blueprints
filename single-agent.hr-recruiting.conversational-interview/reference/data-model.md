# Data model — conversational-interview

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Competency` | `competencyId` | `String` | no | Stable id within the role. |
| | `name` | `String` | no | Short display name. |
| | `description` | `String` | no | What this competency covers. |
| `RoleDefinition` | `roleId` | `String` | no | Stable id for the role. |
| | `roleTitle` | `String` | no | Human-readable role name. |
| | `competencies` | `List<Competency>` | no | Ordered list; at least 1. |
| | `targetTurnCount` | `int` | no | Planned number of Q&A rounds. |
| | `gradingNote` | `String` | no | Optional guidance for the conductor (may be empty string). |
| `SessionRequest` | `sessionId` | `String` | no | UUID minted by `SessionEndpoint`. |
| | `candidateHandle` | `String` | no | Anonymised candidate identifier. |
| | `role` | `RoleDefinition` | no | The role being interviewed for. |
| | `startedBy` | `String` | no | Coordinator identifier. |
| | `startedAt` | `Instant` | no | When the endpoint received the request. |
| `SubmittedAnswer` | `turnIndex` | `String` | no | Which turn this answers. |
| | `rawAnswer` | `String` | no | Candidate's verbatim answer. Audit-only after screening. |
| | `answeredAt` | `Instant` | no | When the endpoint received the answer. |
| `ScreenedAnswer` | `turnIndex` | `String` | no | Matches the `SubmittedAnswer.turnIndex`. |
| | `redactedAnswer` | `String` | no | Protected-category spans replaced with typed tokens. |
| | `protectedCategoriesFound` | `List<String>` | no | e.g. `["health","religion"]`. Empty list if none. |
| `InterviewTurn` | `turnIndex` | `String` | no | Sequential turn number (String for JSON compatibility). |
| | `question` | `String` | no | The question to present to the candidate. |
| | `competencyId` | `String` | no | MUST match a `competencyId` in the role definition. |
| | `sessionComplete` | `boolean` | no | True when all competencies are covered and target reached. |
| | `generatedAt` | `Instant` | no | When the agent returned the turn. |
| `TurnRecord` | `turnIndex` | `String` | no | Matches `InterviewTurn.turnIndex`. |
| | `turn` | `InterviewTurn` | no | The agent-generated question. |
| | `submittedAnswer` | `Optional<SubmittedAnswer>` | yes | Populated after the candidate answers. |
| | `screenedAnswer` | `Optional<ScreenedAnswer>` | yes | Populated after the sanitizer screens the answer. |
| `InterviewSession` | `sessionId` | `String` | no | — |
| | `request` | `Optional<SessionRequest>` | yes | Populated after `SessionStarted`. |
| | `turns` | `List<TurnRecord>` | no | Empty until first `TurnConducted`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionStarted` emitted. |
| | `completedAt` | `Optional<Instant>` | yes | Populated on `SessionCompleted`. |

Every nullable field on `InterviewSession` and `TurnRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SessionStatus`: `OPEN`, `ANSWER_RECEIVED`, `ANSWER_SCREENED`, `CONDUCTING`, `COMPLETED`, `FAILED`.

## Events (`InterviewSessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionStarted` | `request: SessionRequest` | → OPEN |
| `AnswerSubmitted` | `answer: SubmittedAnswer` | → ANSWER_RECEIVED |
| `AnswerScreened` | `screened: ScreenedAnswer` | → ANSWER_SCREENED |
| `TurnConducted` | `turn: InterviewTurn` | → CONDUCTING |
| `SessionCompleted` | `completedAt: Instant` | → COMPLETED (terminal happy) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `InterviewSession.initial("")` with all `Optional` fields as `Optional.empty()`, `turns = Collections.emptyList()`, and `status = OPEN`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `InterviewSession` minus `TurnRecord.submittedAnswer.rawAnswer` (the audit entity keeps that field). The UI fetches the raw answer on demand via `GET /api/sessions/{id}` and reads `turns[n].submittedAnswer.rawAnswer` from the JSON.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`InterviewTasks.java`)

```java
public final class InterviewTasks {
  public static final Task<InterviewTurn> CONDUCT_TURN = Task
      .name("Conduct interview turn")
      .description("Read the role definition and conversation history and produce the next InterviewTurn question")
      .resultConformsTo(InterviewTurn.class);

  private InterviewTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
