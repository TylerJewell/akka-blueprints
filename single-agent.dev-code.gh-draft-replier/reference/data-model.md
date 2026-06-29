# Data model — gitty

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ThreadRef` | `url` | `String` | no | Full GitHub URL, e.g. `https://github.com/owner/repo/issues/42`. |
| | `repoSlug` | `String` | no | `"owner/repo"` extracted from the URL. |
| | `threadNumber` | `long` | no | Issue or PR number. |
| | `kind` | `ThreadKind` | no | `ISSUE` or `PULL_REQUEST`. |
| `QueueRequest` | `draftId` | `String` | no | UUID minted by `DraftEndpoint`. |
| | `thread` | `ThreadRef` | no | Parsed from the submitted URL. |
| | `contextNote` | `String` | no | Maintainer's optional hint (may be empty string). |
| | `queuedBy` | `String` | no | User identifier. |
| | `queuedAt` | `Instant` | no | When the endpoint received the request. |
| `FetchedThread` | `title` | `String` | no | Issue or PR title from GitHub. |
| | `body` | `String` | no | Original post body. |
| | `comments` | `List<String>` | no | Chronological comment bodies. |
| | `commentCount` | `int` | no | Total number of comments fetched. |
| | `fetchedAt` | `String` | no | ISO-8601 timestamp from the fetch. |
| `DraftReply` | `text` | `String` | no | The draft reply text, ≤ 800 characters. |
| | `tone` | `String` | no | One of `{helpful, closing, asking-for-info, acknowledging}`. |
| | `references` | `List<String>` | no | Concrete thread elements the draft cites. |
| | `draftedAt` | `Instant` | no | When the agent returned. |
| `MaintainerAction` | `actorId` | `String` | no | Who approved or discarded. |
| | `editedText` | `String` | no | Approved text (original or edited); empty string on discard. |
| | `actedAt` | `Instant` | no | When the action was taken. |
| `Draft` (entity state) | `draftId` | `String` | no | — |
| | `request` | `Optional<QueueRequest>` | yes | Populated after `ThreadQueued`. |
| | `thread` | `Optional<FetchedThread>` | yes | Populated after `ThreadFetched`. |
| | `draft` | `Optional<DraftReply>` | yes | Populated after `DraftProduced`. |
| | `action` | `Optional<MaintainerAction>` | yes | Populated after `DraftApproved` or `DraftDiscarded`. |
| | `status` | `DraftStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ThreadQueued` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Draft` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ThreadKind`: `ISSUE`, `PULL_REQUEST`.
`DraftStatus`: `QUEUED`, `THREAD_FETCHED`, `DRAFTING`, `PENDING_REVIEW`, `APPROVED`, `DISCARDED`, `FAILED`.

## Events (`DraftEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ThreadQueued` | `request` | → QUEUED |
| `ThreadFetched` | `thread` | → THREAD_FETCHED |
| `DraftingStarted` | — | → DRAFTING |
| `DraftProduced` | `draft` | → PENDING_REVIEW |
| `DraftApproved` | `action` | → APPROVED (terminal happy) |
| `DraftDiscarded` | `action` | → DISCARDED (terminal happy) |
| `DraftFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Draft.initial("")` with all `Optional` fields as `Optional.empty()` and `status = QUEUED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`DraftRow` mirrors `Draft` minus `thread.comments` (the full comment list is large; the UI fetches it on demand via `GET /api/drafts/{id}`). The UI reads `thread.commentCount` and `thread.title` from the view row for the card summary.

The view declares ONE query: `getAllDrafts: SELECT * AS drafts FROM draft_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`DraftTasks.java`)

```java
public final class DraftTasks {
  public static final Task<DraftReply> DRAFT_REPLY = Task
      .name("Draft reply")
      .description("Read the attached GitHub thread and produce a DraftReply for the maintainer")
      .resultConformsTo(DraftReply.class);

  private DraftTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
