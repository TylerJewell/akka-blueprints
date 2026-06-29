# Data model — gitwiki

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PageUpdateRequest` | `updateId` | `String` | no | UUID minted by `WikiEndpoint`. |
| | `pagePath` | `String` | no | Repository-relative path, e.g. `wiki/getting-started.md`. |
| | `pageTitle` | `String` | no | Human-readable page title for UI display. |
| | `currentBody` | `String` | no | Full markdown body of the page at submission time. |
| | `requestedChanges` | `String` | no | Plain-language description of the edit to apply. |
| | `author` | `String` | no | Display name of the submitting user. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `CommitDraft` | `editedBody` | `String` | no | Full page body after the agent applied changes. |
| | `commitMessage` | `String` | no | ≤ 72 characters; imperative mood. |
| | `diffSummary` | `String` | no | One sentence in past tense describing the change. |
| `PushResult` | `status` | `PushStatus` | no | Enum value. |
| | `commitSha` | `String` | yes | Non-null on `SUCCESS`; null otherwise. |
| | `rejectionReason` | `String` | yes | Non-null on `REJECTED`; null otherwise. |
| | `pushedAt` | `Instant` | no | When the GitHub API call returned. |
| `CommitOutcome` (entity state) | `updateId` | `String` | no | — |
| | `draft` | `Optional<CommitDraft>` | yes | Populated after `CommitReady`. |
| | `pushResult` | `Optional<PushResult>` | yes | Populated after `PushCompleted` or `PushRejected`. |
| | `status` | `UpdateStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `UpdateSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `CommitOutcome` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`PushStatus`: `SUCCESS`, `REJECTED`, `CONFLICT`, `FAILED`.

`UpdateStatus`: `SUBMITTED`, `EDITING`, `COMMIT_READY`, `PUSH_IN_PROGRESS`, `PUSHED`, `PUSH_REJECTED`, `CONFLICT`, `FAILED`.

## Events (`PageEntity`)

| Event | Payload | Transition |
|---|---|---|
| `UpdateSubmitted` | `request: PageUpdateRequest` | → SUBMITTED |
| `EditingStarted` | — | → EDITING |
| `CommitReady` | `draft: CommitDraft` | → COMMIT_READY |
| `PushStarted` | — | → PUSH_IN_PROGRESS |
| `PushCompleted` | `sha: String`, `pushedAt: Instant` | → PUSHED or CONFLICT (based on `PushResult.status`) |
| `PushRejected` | `reason: String`, `pushedAt: Instant` | → PUSH_REJECTED |
| `UpdateFailed` | `reason: String` | → FAILED |

`emptyState()` returns `CommitOutcome.initial("")` with all `Optional` fields as `Optional.empty()`, `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`UpdateRow` mirrors `CommitOutcome` plus `pagePath`, `pageTitle`, and `author` extracted from the `UpdateSubmitted` event payload. The `PageUpdateRequest.currentBody` field is **not** included in the view — the UI does not display the full original body.

The view declares ONE query: `getAllUpdates: SELECT * AS updates FROM page_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`WikiTasks.java`)

```java
public final class WikiTasks {
  public static final Task<CommitDraft> EDIT_PAGE = Task
      .name("Edit wiki page")
      .description("Apply the requested changes to the current page body and produce a CommitDraft")
      .resultConformsTo(CommitDraft.class);

  private WikiTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
