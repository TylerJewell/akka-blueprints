# Data model — memory-bank

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MemoryRequest` | `memoryId` | `String` | no | UUID minted by `MemoryEndpoint`. |
| | `operation` | `MemoryOperation` | no | `REMEMBER` or `RECALL`. |
| | `namespace` | `MemoryNamespace` | no | `PERSONAL_ASSISTANT`, `PROJECT_NOTES`, or `CUSTOM`. |
| | `rawContent` | `String` | no | Pre-sanitization content body. Audit-only. |
| | `submittedBy` | `String` | no | User or agent identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedContent` | `redactedContent` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","account-id"]`. |
| `MemoryEntry` | `entryId` | `String` | no | References a stored memoryId. |
| | `sanitizedContent` | `String` | no | The stored redacted content. |
| | `tags` | `List<String>` | no | Agent-extracted tags. |
| | `namespace` | `MemoryNamespace` | no | The entry's namespace. |
| | `storedAt` | `Instant` | no | When the entry reached `STORED`. |
| `StoreConfirmation` | `memoryId` | `String` | no | Echo of the submitted memoryId. |
| | `acknowledgement` | `String` | no | One-sentence confirmation from the agent. |
| | `tags` | `List<String>` | no | Extracted by the agent; 2–5 items. |
| | `confirmedAt` | `Instant` | no | When the agent returned. |
| `RecallMatch` | `memoryId` | `String` | no | References the matching stored entry. |
| | `sanitizedContent` | `String` | no | The matching entry's stored content. |
| | `tags` | `List<String>` | no | Tags on the matching entry. |
| | `relevanceScore` | `float` | no | 0.0–1.0; higher is more relevant. |
| | `storedAt` | `Instant` | no | When the matching entry was stored. |
| `RecallResult` | `queryContent` | `String` | no | Echo of the recall query. |
| | `matches` | `List<RecallMatch>` | no | Ranked list; may be empty. |
| | `recalledAt` | `Instant` | no | When the agent returned. |
| `Memory` (entity state) | `memoryId` | `String` | no | — |
| | `request` | `Optional<MemoryRequest>` | yes | Populated after `MemoryEntrySubmitted`. |
| | `sanitized` | `Optional<SanitizedContent>` | yes | Populated after `MemoryEntrySanitized`. |
| | `confirmation` | `Optional<StoreConfirmation>` | yes | Populated after `MemoryEntryStored` (REMEMBER). |
| | `recallResult` | `Optional<RecallResult>` | yes | Populated after `MemoryEntryStored` (RECALL). |
| | `status` | `MemoryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MemoryEntrySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Memory` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`MemoryOperation`: `REMEMBER`, `RECALL`.
`MemoryNamespace`: `PERSONAL_ASSISTANT`, `PROJECT_NOTES`, `CUSTOM`.
`MemoryStatus`: `SUBMITTED`, `SANITIZED`, `STORING`, `STORED`, `FAILED`.

## Events (`MemoryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MemoryEntrySubmitted` | `request` | → SUBMITTED |
| `MemoryEntrySanitized` | `sanitized` | → SANITIZED |
| `MemoryStoringStarted` | — | → STORING |
| `MemoryEntryStored` | `confirmation` (REMEMBER) or `recallResult` (RECALL) | → STORED (terminal happy) |
| `MemoryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Memory.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`MemoryRow` mirrors `Memory` minus `request.rawContent` (the audit log keeps that). The UI fetches the raw content on demand via `GET /api/memories/{id}` and reads `request.rawContent` from the JSON.

The view declares ONE query: `getAllMemories: SELECT * AS memories FROM memory_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`MemoryTasks.java`)

```java
public final class MemoryTasks {
  public static final Task<StoreConfirmation> REMEMBER_CONTENT = Task
      .name("Remember content")
      .description("Store the sanitized content with extracted tags and return a StoreConfirmation")
      .resultConformsTo(StoreConfirmation.class);

  public static final Task<RecallResult> RECALL_CONTENT = Task
      .name("Recall content")
      .description("Search stored memory entries for the query and return a ranked RecallResult")
      .resultConformsTo(RecallResult.class);

  private MemoryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
