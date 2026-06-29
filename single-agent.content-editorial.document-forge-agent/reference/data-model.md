# Data model — document-forge-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ForgeRequest` | `forgeId` | `String` | no | UUID minted by `ForgeEndpoint`. |
| | `prompt` | `String` | no | User-supplied description of the document to generate. |
| | `documentType` | `String` | no | One of `BUSINESS_MEMO`, `TECHNICAL_BRIEF`, `MARKETING_COPY`, `PROJECT_PROPOSAL`, `CUSTOM`. |
| | `styleTemplate` | `String` | no | One of `FORMAL`, `CASUAL`, `TECHNICAL`, `EXECUTIVE_SUMMARY`. |
| | `requestedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `DocumentOutput` | `filename` | `String` | no | Safe relative filename chosen by the agent. |
| | `content` | `String` | no | The generated document body. |
| | `formatHint` | `String` | no | One of `markdown`, `plain-text`, `html`. |
| | `wordCount` | `int` | no | Word count computed from content at time of write. |
| `AuditEntry` | `score` | `int` | no | 1–5 from `ForgeAuditor`. |
| | `assessment` | `String` | no | One-line quality assessment. |
| | `flaggedPatterns` | `List<String>` | no | Patterns found (empty list if clean). |
| | `auditedAt` | `Instant` | no | When `ForgeAuditor` finished. |
| `ForgeResult` | `outputFilename` | `String` | no | Matches the filename passed to `write_document`. |
| | `documentContent` | `String` | no | Matches the content passed to `write_document`. |
| | `agentRationale` | `String` | no | 2–4 sentences from the agent explaining its choices. |
| | `forgedAt` | `Instant` | no | When the agent returned. |
| `ForgeRun` (entity state) | `forgeId` | `String` | no | — |
| | `request` | `Optional<ForgeRequest>` | yes | Populated after `ForgeSubmitted`. |
| | `output` | `Optional<DocumentOutput>` | yes | Populated after `ForgeCompleted`. |
| | `audit` | `Optional<AuditEntry>` | yes | Populated after `ForgeAudited`. |
| | `status` | `ForgeStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ForgeSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ForgeRun` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ForgeStatus`: `SUBMITTED`, `FORGING`, `FORGE_COMPLETED`, `AUDITED`, `FAILED`.
`DocumentType`: `BUSINESS_MEMO`, `TECHNICAL_BRIEF`, `MARKETING_COPY`, `PROJECT_PROPOSAL`, `CUSTOM`.
`StyleTemplate`: `FORMAL`, `CASUAL`, `TECHNICAL`, `EXECUTIVE_SUMMARY`.

## Events (`ForgeEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ForgeSubmitted` | `request` | → SUBMITTED |
| `ForgeStarted` | — | → FORGING |
| `ForgeCompleted` | `output` | → FORGE_COMPLETED |
| `ForgeAudited` | `audit` | → AUDITED (terminal happy) |
| `ForgeFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ForgeRun.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ForgeRow` mirrors `ForgeRun`. No fields are stripped — the forge prompt and output content are appropriate for the UI and no raw-vs-redacted split applies in this domain.

The view declares ONE query: `getAllForges: SELECT * AS forges FROM forge_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ForgeTasks.java`)

```java
public final class ForgeTasks {
  public static final Task<ForgeResult> FORGE_DOCUMENT = Task
      .name("Forge document")
      .description("Generate a structured document from the provided prompt and style template, then write it via the write_document tool")
      .resultConformsTo(ForgeResult.class);

  private ForgeTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
