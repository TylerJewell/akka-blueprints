# Data model — bug-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `BugReport` | `bugId` | `String` | no | UUID minted by `BugEndpoint`. |
| | `title` | `String` | no | One-line bug title. |
| | `description` | `String` | no | Full bug description, including stack trace or reproduction steps. |
| | `priority` | `Priority` | no | Enum value: LOW / MEDIUM / HIGH / CRITICAL. |
| | `component` | `String` | no | System component or service name the bug belongs to. |
| | `reportedBy` | `String` | no | User identifier. |
| | `reportedAt` | `Instant` | no | When the endpoint received the report. |
| `TicketMetadata` | `ticketKey` | `String` | no | Deterministically minted key, e.g. `PROJ-1042`. |
| | `assignee` | `String` | no | Team member assigned from a seeded list. |
| | `labels` | `List<String>` | no | Includes priority label, component label, and `"bug"`. |
| | `project` | `String` | no | Project prefix extracted from `ticketKey`. |
| | `fetchedAt` | `Instant` | no | When `TicketSyncConsumer` built the metadata. |
| `SearchResult` | `query` | `String` | no | The query string that produced this result. |
| | `source` | `String` | no | `"web"` or `"ticket-history"`. |
| | `snippet` | `String` | no | Text snippet from the result. May be empty string. |
| | `url` | `String` | no | Source URL or ticket reference. |
| `Resolution` | `status` | `ResolutionStatus` | no | Enum value. |
| | `resolutionBody` | `String` | no | At least 20 characters. Root cause + fix or recommendation. |
| | `confidenceLevel` | `ConfidenceLevel` | no | Enum value. |
| | `evidence` | `List<SearchResult>` | no | Search results the agent cited. May be empty list. |
| | `resolvedAt` | `Instant` | no | When the agent returned the resolution. |
| `Bug` (entity state) | `bugId` | `String` | no | — |
| | `report` | `Optional<BugReport>` | yes | Populated after `BugSubmitted`. |
| | `ticket` | `Optional<TicketMetadata>` | yes | Populated after `TicketEnriched`. |
| | `resolution` | `Optional<Resolution>` | yes | Populated after `ResolutionRecorded`. |
| | `status` | `BugStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `BugSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Bug` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Priority`: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.
`ResolutionStatus`: `FIXED`, `NEEDS_MORE_INFO`, `WONT_FIX`, `DUPLICATE`.
`ConfidenceLevel`: `HIGH`, `MEDIUM`, `LOW`.
`BugStatus`: `SUBMITTED`, `ENRICHED`, `INVESTIGATING`, `RESOLVED`, `FAILED`.

## Events (`BugEntity`)

| Event | Payload | Transition |
|---|---|---|
| `BugSubmitted` | `report` | → SUBMITTED |
| `TicketEnriched` | `ticket` | → ENRICHED |
| `InvestigationStarted` | — | → INVESTIGATING |
| `ResolutionRecorded` | `resolution` | → RESOLVED |
| `BugFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Bug.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`BugRow` mirrors `Bug` and additionally includes a `toolCallTrace: List<ToolCallEntry>` field. Each `ToolCallEntry` carries `tool: String`, `arguments: Map<String,Object>`, `result: String`, `outcome: String` (`"PASSED"` or `"REJECTED"`), and `calledAt: Instant`. The trace is populated from the agent's tool invocations recorded during `investigateStep`.

The view declares ONE query: `getAllBugs: SELECT * AS bugs FROM bug_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`BugTasks.java`)

```java
public final class BugTasks {
  public static final Task<Resolution> RESOLVE_BUG = Task
      .name("Resolve bug")
      .description("Investigate the bug using available tools and write a resolution to the ticket")
      .resultConformsTo(Resolution.class);

  private BugTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
