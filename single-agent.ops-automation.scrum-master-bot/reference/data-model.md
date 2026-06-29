# Data model — scrum-master-bot

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TeamMember` | `memberId` | `String` | no | Stable id supplied by the user (e.g. `"alice"`). |
| | `displayName` | `String` | no | Human-readable name for the UI. |
| | `role` | `String` | no | Team role (e.g. `"Senior Engineer"`). |
| `SprintContext` | `sprintId` | `String` | no | Stable sprint identifier. |
| | `sprintName` | `String` | no | Human-readable sprint name. |
| | `sprintNumber` | `int` | no | Sprint sequence number. |
| | `teamName` | `String` | no | Team label. |
| | `members` | `List<TeamMember>` | no | Roster (1–N). |
| | `authorizedTicketIds` | `List<String>` | no | Ticket ids the agent may write to. |
| | `sprintStartAt` | `Instant` | no | Sprint start timestamp. |
| | `sprintEndAt` | `Instant` | no | Sprint end timestamp. |
| `MemberUpdate` | `memberId` | `String` | no | MUST equal a member in the roster. |
| | `yesterday` | `String` | no | What they did yesterday. |
| | `today` | `String` | no | What they plan today. |
| | `blocker` | `Optional<String>` | yes | Non-null if the member reported a blocker. |
| `TicketUpdate` | `ticketId` | `String` | no | MUST be in `authorizedTicketIds`. |
| | `comment` | `String` | no | Standup note posted to the ticket. |
| | `newStatus` | `Optional<String>` | yes | Non-null only if the member changed ticket status. |
| `StandupSummary` | `sessionOutcome` | `SessionOutcome` | no | Enum value. |
| | `summaryText` | `String` | no | 2–4 sentence paragraph. |
| | `memberUpdates` | `List<MemberUpdate>` | no | One entry per team member. |
| | `ticketUpdates` | `List<TicketUpdate>` | no | One entry per ticket posted to. |
| | `nextActions` | `List<String>` | no | 2–4 actionable verb-phrases. |
| | `conductedAt` | `Instant` | no | When the agent returned. |
| `PostResult` | `ticketsPosted` | `int` | no | Count of successfully posted tickets. |
| | `skippedTicketIds` | `List<String>` | no | Ids rejected by the guardrail or failed to post. |
| | `postedAt` | `Instant` | no | When `postStep` finished. |
| `StandupSession` (entity state) | `sessionId` | `String` | no | — |
| | `sprintContext` | `Optional<SprintContext>` | yes | Populated after `SprintContextAttached`. |
| | `summary` | `Optional<StandupSummary>` | yes | Populated after `SummaryRecorded`. |
| | `postResult` | `Optional<PostResult>` | yes | Populated after `UpdatesPosted`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SprintActivated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `StandupSession` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SessionOutcome`: `ON_TRACK`, `AT_RISK`, `BLOCKED`.
`SessionStatus`: `COLLECTING`, `RUNNING`, `SUMMARY_READY`, `POSTED`, `FAILED`.

## Events (`StandupEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SprintActivated` | `sprintId, teamName, members, authorizedTicketIds` | → COLLECTING |
| `SprintContextAttached` | `sprintContext` | stays COLLECTING |
| `StandupStarted` | — | → RUNNING |
| `SummaryRecorded` | `summary` | → SUMMARY_READY |
| `UpdatesPosted` | `postResult` | → POSTED (terminal happy) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `StandupSession.initial("")` with all `Optional` fields as `Optional.empty()` and `status = COLLECTING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `StandupSession`. The UI fetches full session detail via `GET /api/standups/{id}`.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM standup_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`StandupTasks.java`)

```java
public final class StandupTasks {
  public static final Task<StandupSummary> CONDUCT_STANDUP = Task
      .name("Conduct standup")
      .description("Run the daily standup for the sprint team and return a StandupSummary")
      .resultConformsTo(StandupSummary.class);

  private StandupTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
